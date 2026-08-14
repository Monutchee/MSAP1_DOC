# FIXRP00001 — Meter Sequence Gaps and the Historian WAL Checkpoint Failure

| | |
|---|---|
| Incident dates | 2026-08-13 (record loss), 2026-08-14 (startup delay, silent history loss) |
| Affected device | MSAP1 development unit, `/data` on removable SD via USB reader |
| Fix delivered | 2026-08-14 — `MSAP1_APU` PR #43 (`fix/spool_miss_shot`), `meta-monutchee` PR #51 (`fix/meter-dma-ring-depth`) |
| Verified | 21/21 host unit tests, aarch64 cross build, live-device monitoring |

## 1. Why we spotted the problem

Three symptoms surfaced within two days, each initially looking unrelated:

1. **`meter_sequence_gap` warnings** in the portal event log, 2026-08-13
   between roughly 13:34 and 13:39 local time: `meter record sequence gap:
   expected N, got N+1` from `fpga-acquisition`. The journal showed the true
   extent — **793 events from 16:59:24 to 17:39:01 UTC**, exactly 20 per
   minute, one basic MTR1 record lost out of every 15 (i.e. once per 3 s
   aggregate window), then spontaneous recovery. The interval tracked grid
   frequency rather than wall time, pinning the loss to the MTR2 aggregate
   emission boundary.
2. **The web portal took ~80 s to become reachable after boot**
   (`ERR_CONNECTION_REFUSED` while nginx waited on backend readiness).
3. After the spool was switched to the memory backend as a test
   (2026-08-13 ~20:07 UTC) and the device rebooted, the **historian silently
   stopped persisting new history**: its WAL file stopped receiving writes
   entirely while records demonstrably kept flowing.

## 2. What the problem was

The investigation found one incident with two root causes and, behind them,
three latent design assumptions. All were confirmed with direct evidence, not
inference.

### 2.1 Record loss: kernel DMA ring overrun behind a blocking publish

- The PL was exonerated by register accounting: `RESULT_SEQ (0x24)` equalled
  `AGG_RECORD_COUNT (0x7C) × 15 + AGG_INELIGIBLE (0x84) + blocks in flight`
  exactly. Every block the PL ever produced was accounted for.
- The loss point was the kernel transport. `/dev/msap1-meter`'s cyclic DMA
  ring was **4 records deep** (3-record safe window ≈ 600 ms of tolerated
  consumer stall at the 200 ms cadence); `msap1_dma_catch_up()` silently
  drops the oldest period on overrun, and the meter path — unlike the
  waveform path — never queried `MSAP1_DMA_IOC_TRANSPORT_STATUS`, so
  kernel-side loss was indistinguishable from PL loss.
- The consumer stalled because `MeterRecordIngestor::accept()` publishes
  **every record through a synchronous IPC round-trip** to
  `msap1-meter-stream`, whose `DurableMeterSpool` committed each record with
  `PRAGMA synchronous=FULL` — a full fsync on `/data` per record. `/data` is
  a removable consumer SD card; during a ~40-minute write-latency episode
  (an internal flash housekeeping window, aggravated by the WAL problem
  below) each publish exceeded the 600 ms ring budget once per aggregate
  boundary — the burst point where basic block 15, the aggregate record, and
  the historian's own persistent commit coincide. Result: exactly one
  record lost every 3 s, and "self-recovery" when the SD latency episode
  ended.

### 2.2 Startup delay and disk churn: WAL checkpoints never completed

The historian's database directory told the story:

```
historian.sqlite3        28 KB   (mtime frozen days earlier)
historian.sqlite3-wal   1.39 GB  (growing ~8 KB/s, ~700 MB/day)
```

Decoding the WAL-index shared memory showed **337,864 frames with
`nBackfill = 0`** — not a single frame had ever been copied back into the
main database — while the true content was only ~5,000 pages (~20 MB)
rewritten hundreds of thousands of times. Every boot paid a full WAL
recovery scan over that file before the historian reported ready, which is
the ~80 s portal delay.

Root cause: in `MeterHistoryStore::append()`
(`common/msap1/meter/history/meter_history.cpp`), the `find` statement
(`SELECT id FROM measurement_blocks WHERE stream_cursor=?`) was stepped once
to `SQLITE_ROW` and neither reset nor finalized before
`transaction.commit()`. SQLite runs its WAL auto-checkpoint at COMMIT and
**aborts whenever the committing connection still has an active statement**
— every commit, deterministically. This was proven by an A/B reproduction
against the image's exact SQLite 3.45.3: 20,000 identical commits with the
statement active at COMMIT left a 4 KB main DB and a 505 MB WAL; finalizing
the statement first produced a 2 MB main DB with the WAL stable at 4 MB.
The same latent pattern existed in the spool's `publish()` duplicate path,
`acknowledge()`, and `prune()`.

### 2.3 Latent hazards of a volatile spool (found during the fix design)

Switching the spool to the memory backend — the correct cure for 2.1 — was
unsafe as a bare config flip, and the test device demonstrated all three
hazards live after its 2026-08-14 reboot:

1. **Cursor reuse.** A fresh `:memory:` spool restarts AUTOINCREMENT at 1.
   The historian dedups on `stream_cursor INTEGER NOT NULL UNIQUE` +
   `INSERT OR IGNORE` and keeps persisted clear floors keyed by cursor, so
   reused cursors silently discarded every new record into the persistent
   datasets after every reboot. (Observed: historian WAL untouched for
   hours while records flowed.)
2. **Consumer wedge.** The `consumers` table vanishes with the meter-stream
   process; the historian's consume loop retried forever without
   re-registering, and the unacknowledged spool then grew without bound
   (no byte cap by default) toward an OOM kill that would also tear down
   acquisition, since `publish()` failures propagate.
3. **Unreported loss.** An empty spool reports `oldest_cursor = 0`, which
   defeated the backfill-completeness heuristic `oldest_cursor > 1` — loss
   at power-cut would pass as a clean, complete rebuild.

## 3. How we fixed it

All application changes are in `MSAP1_APU` commit `b0d221a`
(PR #43); kernel/layer changes in `meta-monutchee` commits `beff99b` and
`be504ce` (PR #51).

### 3.1 WAL checkpoint fix

Reset every row-active statement before its connection commits:
`find.reset()` in `MeterHistoryStore::append()`, plus the same-class resets
in `DurableMeterSpool::publish()` (duplicate path), `acknowledge()`, and
`prune()`. Each site carries a comment stating the constraint (commit-time
auto-checkpoints abort while the connection has an active statement).

### 3.2 Memory-spool hardening, then the backend switch

- **Cursor lease** (`spool.sqlite3.cursor-lease`): at spool construction the
  cursor space is seeded into `sqlite_sequence` from
  `max(persisted lease, database maximum, first-boot wall-clock bootstrap)`
  and a reservation of 2³¹ cursors is written back atomically
  (temp + fsync + rename). Renewal runs on the acknowledge/prune path —
  never the producer's publish path. Cursors are now strictly monotonic
  across restarts of any backend; the millisecond wall-clock bootstrap makes
  upgrades collision-free against deployed count-based cursors.
- **Hard byte cap in `publish()`** with drop-oldest eviction (the newest
  record always survives), an `dropped_unacknowledged_records` counter, and
  a rate-limited `spool_records_dropped` warning. Enforcement lives in
  `publish()` because `prune()` only runs from the acknowledge handler —
  absent in exactly the wedged-consumer scenario. Bounded, reported loss
  replaces unbounded memory growth.
- **Historian re-registration**: the consume-loop error path now re-asserts
  `register_consumer("historian")` (idempotent — a live registration keeps
  its acknowledged cursor) before its retry sleep.
- **Truthful backfill detection**: `backfill_is_incomplete()` replaces the
  `oldest_cursor > 1` heuristic, comparing the stream's new
  `session_start_cursor` against the historian's persisted high-water
  (`persisted_stream_high_water()`); an empty volatile spool can no longer
  masquerade as complete coverage.
- **Status plumbing**: `StreamStatus` gains `session_start_cursor` and
  `dropped_unacknowledged_records` (IPC protocol v2, additive fields; all
  peers ship in one image and the decoder's `require_finished()` makes any
  mix fail loudly). Both fields surface in the developer database API.
- **Defaults**: `factory-defaults.json` and the mirrored
  `database_settings.hpp` defaults now ship
  `spool: backend=memory, maximum_bytes=32 MiB,
  volatile_spool_acknowledged=true`. Existing devices keep their persisted
  settings until an operator applies the new policy (live migration via
  `apply_storage_policy` preserves records and consumer cursors).

### 3.3 Why the memory backend is better than disk for the spool

The choice is not a durability-versus-safety trade-off, because the spool's
durability never protected the archive. The billing-grade measurements
(150/180-cycle, 10-minute, 2-hour datasets) are durable in the **historian's
own** `synchronous=FULL` SQLite database regardless of the spool backend. A
persistent spool only ever protected the *handoff window*: raw records
published but not yet projected by the historian (milliseconds in normal
operation, at most the 1 h retention window), plus the ability to rebuild
the deliberately-volatile `basic` dataset after a reboot. Against that
narrow benefit, the persistent spool carried three structural costs:

1. **Hot-path tail latency.** The producer's publish is a synchronous
   round-trip inside the acquisition loop. A memory commit is microseconds
   with essentially no variance; an fsync'd commit is milliseconds
   *typically* but unbounded in the tail on consumer flash — SD/eMMC
   garbage-collection pauses reach hundreds of milliseconds to seconds,
   which is precisely what overran the DMA ring in this incident. Even with
   the ring deepened to 8 records (~1.4 s budget), a persistent spool keeps
   a storage-latency dependency inside the one loop that must never stall.
   The memory backend removes the failure class, not just this instance.
2. **Flash endurance, which feeds back into (1).** The persistent spool
   wrote ~5.3 records/s, each an fsync'd commit — roughly 700 MB/day,
   ~250 GB/year of small random synchronous writes, continuously, onto
   removable consumer media with limited endurance. Worn flash develops
   longer and more frequent housekeeping pauses, so the persistent spool
   was actively manufacturing the latency spikes that break it: a slow
   positive-feedback loop toward exactly this incident. The memory spool
   eliminates that write stream entirely; the remaining `/data` writes are
   the historian's aggregate-cadence commits at a small fraction of the
   volume, with checkpointing now functional.
3. **Disproportionate resource cost.** The steady-state spool is ~8 MB of
   RAM (hard-capped at 32 MiB with counted, reported eviction) against a
   continuous disk-wear and latency liability. On this hardware RAM is
   plentiful and `/data` is the scarcest, most failure-prone resource.

What is genuinely given up: up to one hour of *unprojected raw records* at a
power cut, and the post-reboot rebuild window for the volatile `basic`
history — both now truthfully reported through `backfill_incomplete` and
`session_start_cursor` instead of passing silently. Disk would only be the
right choice again if a requirement appeared for at-least-once delivery of
raw records across power cuts (audit-grade raw retention), or on a variant
with scarce RAM and soldered industrial eMMC rated for the write load.

### 3.4 Kernel and attribution changes

- **Meter DMA ring deepened 4 → 8 records** (`msap1_dma_meter.c`). Ring
  depth adds no steady-state latency — each completed period wakes the
  reader immediately — it trades first-record latency after capture start
  (ring × cadence, a Xilinx cyclic-DMA first-callback property) against
  stall tolerance: 0.8 s / 0.6 s before, 1.6 s / 1.4 s now.
- **Loss attribution at the source**: the acquisition meter path now queries
  the kernel's transport-overrun counter and logs the delta with every
  `meter_sequence_gap` event (`MNC_TRANSPORT_OVERRUNS`,
  `MNC_TRANSPORT_OVERRUN_DELTA`). A future gap states on its face whether it
  was a kernel ring overrun (consumer stall) or upstream/PL loss — the
  distinction that cost this investigation most of a day.
- **Recipe hygiene** (`msap1-dma_1.0.bb`): sources moved out of the
  `${WORKDIR}` root (`S = "${WORKDIR}/src"`, `;subdir=src` on each
  `SRC_URI` entry). `base.bbclass` wipes `${S}` on unpack only when
  `S != WORKDIR`, so the old layout never cleaned the tree between builds —
  the cause of the pseudo "inode mismatch" warning storms after every
  driver source edit.

## 4. How we tested the fix

### 4.1 Reproduction before the fix (evidence, not conjecture)

- A/B WAL reproduction against SQLite 3.45.3 from the yocto native sysroot
  (section 2.2): statement-active-at-commit grows the WAL unboundedly with a
  frozen main DB; statement-finalized-before-commit checkpoints normally.
- Live WAL-index header decode on the device: `mxFrame = 337,864`,
  `nBackfill = 0`, content 5,027 pages.
- Block-accounting register reconciliation proving zero PL-side loss.

### 4.2 Unit tests (all in `tests/meter_stream_test.cpp`)

| Test | Guards |
|---|---|
| `historian_wal_stays_bounded_under_sustained_appends` | The WAL bug itself: 2,000 fsync'd appends must keep the WAL < 8 MB and grow the main DB. Fails on the pre-fix append path (~50 MB WAL, empty main DB). |
| `memory_spool_cursors_survive_restart` | Lease monotonicity across `:memory:` restarts; lease file creation; corrupt-lease fallback stays monotonic for a persistent spool via its database maximum. |
| `publish_enforces_hard_byte_cap` | Bounded record count with no acknowledgements, eviction counter grows, newest record survives, survivors stay cursor-ordered. |
| `register_consumer_preserves_acknowledged_cursor` | The idempotency the re-registration fix depends on. |
| `backfill_predicate_reports_lost_coverage` | Truth table: virgin+fresh complete, in-session pruning incomplete, post-coverage session incomplete, empty post-coverage spool incomplete (the case the old heuristic passed), covering session complete. |

Full suite: **21/21 pass** on the native host build; the aarch64 cross build
compiles clean. The changed kernel module and recipe were rebuilt with
bitbake (cleansstate + full recipe run, 1,029 tasks succeeded), and the
recipe regression scenario — edit `msap1_dma_meter.c`, rebuild — produced
**zero** pseudo inode-mismatch warnings.

### 4.3 Live device verification (172.30.19.99, fresh `/data/mnc/database`)

- **Checkpoint behaviour, monitored through the first threshold crossing**
  (30 s samples):

  ```
  14:38–14:44   WAL 0.98 → 3.96 MB     main DB 28 KB (unchanged)
  14:45:13      WAL frozen at 4,120,032 B   main DB 28 KB → 90,112 B
  14:45–14:47   WAL byte-identical          ingest uninterrupted (41 → 236 blocks)
  ```

  4,120,032 B = 1000 frames × (4,096-byte page + 24-byte header) — the exact
  SQLite auto-checkpoint threshold. The WAL is now a reused ring; its file
  size is expected to sit at ~4.12 MB permanently. The old code never
  backfilled one frame.
- **Volatile spool correctness**: portal shows durability "Volatile", a
  gap-free cursor range whose span equals the record count, historian lag 0,
  and the new backfill banner truthfully reporting that pre-session records
  are unavailable — the loss that was previously silent.
- **Startup**: with the runaway WAL gone, the historian (and therefore the
  portal) no longer pays the ~80 s recovery scan at boot.

## 5. Conclusion

A single field symptom — one lost meter record every three seconds for forty
minutes — unwound into three independent defects that had been masking each
other: a synchronous fsync-per-record publish sitting on the acquisition hot
path, a one-line statement-lifetime bug that had silently disabled SQLite
checkpointing since the historian first shipped, and a set of restart-safety
assumptions that only held while the spool happened to be persistent.

The durable lessons:

1. **Keep storage latency off hot paths.** Any synchronous disk write in the
   acquisition loop is a record-loss bug waiting for a slow-flash day. The
   spool is now memory-backed by default; the durable archive lives in the
   historian, where latency is harmless.
2. **Never commit with a row-active statement.** SQLite's commit-time
   auto-checkpoint silently aborts, and nothing tells you. The regression
   test now enforces the observable outcome (bounded WAL) rather than the
   coding pattern.
3. **Loss must be attributable and reported.** The kernel overrun delta in
   every gap log, the eviction counter, and the truthful backfill flag each
   convert a class of silent failure into a one-line diagnosis.
4. **Volatile state needs an epoch.** Any identifier that consumers persist
   (the stream cursor) must be monotonic across producer restarts; the
   lease file provides that for a few bytes of `/data` per service start.

Open follow-ups, deliberately out of scope here: retention caps for the
persistent historian datasets (defaults are *forever*; ~4–5 GB/year accrues
at the current aggregate rate), and periodic review of
`dropped_unacknowledged_records` in fleet monitoring once devices run the
memory spool at scale.
