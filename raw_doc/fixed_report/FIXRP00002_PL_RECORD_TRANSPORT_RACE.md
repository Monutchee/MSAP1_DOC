# FIXRP00002 — Periodic Meter Sequence Gaps: the PL Record-Transport Race

| | |
|---|---|
| Incident window | 2026-08-13 through 2026-08-15 (four confirmed episodes + one reattributed) |
| Affected device | MSAP1 development unit 172.30.19.99 (Kria K26, `xck26-sfvc784-2LV`) |
| Symptom | One basic (MTR1) record lost per 3 s aggregate window during self-clearing episodes of 6–111 minutes; episode exits lose a burst |
| Root cause (convicted class) | Physical race at the 2,048-bit combinational arbiter→packetizer handoff in the PL record transport |
| Fix | Registered valid/ready output stage in `measurement_record_arbiter` (MSAP1_PL `85ee806`, branch `fix/periodic_seq_gap`) |
| Status | Fix built and simulated; **awaiting 72 h field soak** (the deciding experiment — see §9) |
| Superseded by | **Part II (§13 onward)** — the soak returned *recurrence*; this Part's conclusions are corrected in §19. Read Part I as the contemporaneous record, not the final answer. |
| Companion report | FIXRP00001 (the 2026-08-13 storage-side incident: WAL checkpoint bug, memory spool, kernel ring) |

## 1. Executive summary

Over three days, a development unit exhibited recurring episodes in which
exactly one basic meter record per 150/180-cycle aggregation window
(3 seconds) failed to reach the application processor, while the aggregate
(MTR2) stream stayed perfect. Episodes began and ended spontaneously, left
every drop counter in the system at zero, and ended with a characteristic
burst loss. Three generations of purpose-built instrumentation — kernel
transport attribution, then rejection forensics with raw record dumps, then
a live environmental sampler — progressively eliminated the kernel, the
storage stack, the ADC, the environment, the PL's computation pipeline, and
finally (by exhaustive simulation) the transport RTL's functional behavior.

What remained is a physical race at the one place in the design where 2,048
bits crossed a combinational boundary into capture registers: the record
arbiter's unregistered mux feeding the packetizer. The observed record-level
behavior — bit-perfect duplicated aggregates replacing lost basics, and
occasional hybrid records with one producer's head and the other's zeroed
tail — is precisely what capture-boundary races produce and what no
functional or environmental mechanism could. The fix registers the handoff,
making the race structurally impossible. It passes the entire simulation
suite including a new exhaustive phase-sweep bench, and builds with zero
timing violations.

One suspect could not be exonerated by simulation (the HLS aggregator shim's
event path, upstream of the transport), so this report states its
conclusions with deliberate precision: the fault class is convicted, the
specific electrical mechanism is inferred, and the 72-hour instrumented soak
on the fixed bitstream is the experiment that separates "confirmed" from
"redirect to the shim". Either outcome is fully instrumented.

## 2. How the problem presented

Portal event log, 2026-08-14: runs of `meter record sequence gap: expected
N, got N+1` warnings from `fpga-acquisition`, one every 3 seconds, each gap
exactly one record. Each episode ended with a single larger gap (4–8
records) and then silence. The 2026-08-13 incident (FIXRP00001) had shown
the same surface symptom and was attributed at the time to a storage-latency
/ DMA-ring overrun chain; this report's findings retroactively cast doubt on
that attribution (§8).

### Episode ledger

| Episode | Onset (UTC) | End | Duration | Stuck slot (seq mod 15) | Exit burst | Boot-relative onset |
|---|---|---|---|---|---|---|
| (Aug-13, reattributed) | 08-13 16:59:24 | 17:39:01 | 39.6 min | 13 | 4 (slots 13→1) | — (old image) |
| A | 08-14 17:17:08 | 17:50:56 | 33.8 min | 9 | 8 (71229→71237, slots 9→1) | 3 h 24 m |
| B | 08-14 21:19:17 | 21:25:50 | 6.5 min | 9 | 8 (135699→135707) | 7 h 26 m |
| C | 08-15 09:04:10 | 10:55:03 | 110.9 min | 9 | 8 (199374→199382) | 9 h 14 m |
| D | 08-15 16:13:39 | 17:01:22 | 47.7 min | 9 | 8 (86559→86567) | 4 h 01 m |

Structural constants across every episode: loss rate exactly one basic
record per window; the lost record always at one fixed absolute slot
(≡ 9 mod 15 on the current bitstream, ≡ 13 on the previous one); aggregates
never lost; and the exit burst always spanning **from the stuck slot through
slot 1 of the next window** — `burst = (1 − slot) mod 15 + 1`, which
predicts 8 for slot 9 and 4 for slot 13. That formula fitting the Aug-13
incident (4, slot 13) on a *different bitstream* is the strongest single
piece of evidence that all five events share one mechanism. An early
hypothesis that onsets recur every ~3.4 h died against the ledger: the
boot-relative onsets share nothing.

## 3. Instrumentation, in three generations

The investigation was gated throughout by what the system could *say* about
its own losses. Each generation of instrumentation was built because the
previous one's answer was incomplete — and one was built because its
predecessor's answer was wrong.

**Generation 1 — kernel transport attribution** (APU `fix/spool_miss_shot`,
shipped with FIXRP00001): every `meter_sequence_gap` warning carries the
kernel DMA ring's overrun counter delta (`MSAP1_DMA_IOC_TRANSPORT_STATUS`).
From the first post-deploy episode onward, every gap read **"no kernel
transport overrun"** — the loss was upstream of the kernel, which the
previous incident's tooling could not have shown.

**Generation 2 — configuration-mismatch forensics** (APU `236e1c6`): the
acquisition daemon's only *silent* rejection path was believed to be
`matches_configuration()`; it gained a rate-limited log with the rejected
record's raw header words. Episode C then **falsified the premise**: 2,217
records were swallowed with the counters in perfect lockstep and *zero*
config-mismatch events. The swallow site had to be the *other* silent path
— the stale/out-of-order sequence guard — which the first instrumentation
deliberately did not cover.

**Generation 3 — complete rejection coverage** (APU `0cae2ca`): every
rejection path (stale/out-of-order on both streams, configuration mismatch,
decoder) dumps raw words 0–15 and 60–61, rate-limited with a suppressed
count so a hidden second loss per window can never masquerade as the
one-per-window pattern. From this point no record could leave the system
without leaving evidence. Episode D was captured end-to-end with it.

In parallel, a read-only **margin-watch sampler** ran on the device (1-min
cadence: die temperatures, SYSMON rails, acquisition counters, PL registers)
so the next onset would carry an environmental trace — it caught episode D's
onset within 60 seconds of starting.

## 4. Eliminations, with their evidence

Each of the following was excluded by measurement, not argument:

1. **Kernel DMA transport** — overrun counter zero through every episode
   (Gen-1 logging); ring stale-slot replay refuted separately in episode D:
   a stale ring slot would replay content from exactly `ring_periods` (8)
   records earlier, but the duplicate arrived immediately after its
   original.
2. **Storage / spool / scheduling** — the memory-backed spool (FIXRP00001)
   removed every disk dependency from the hot path; publish is
   microseconds; episodes occurred anyway.
3. **PL computation pipeline** — the block-accounting identity
   `RESULT_SEQ = aggregates×15 + ineligible + in-flight` reconciled
   **exactly** at every check, including across episode exits;
   `RESULT_SEQ` advanced at precisely 5.000 blocks/s through episodes;
   grid lock never dropped (`AGG_INELIGIBLE` pinned at the single boot
   artifact); aggregation never reset.
4. **ADC and its SPI health issues** — the AD7771's data path showed zero
   header errors / FIFO overflows throughout; the chronic SPI *status-poll*
   flakiness (a separate nuisance, see
   `MSAP1_RPU/docs/future_plan/ADC_HEALTH_POLL_HARDENING.md`) decorrelated
   cleanly: flaps fired during perfectly healthy stream periods and
   episodes ran without flaps at their boundaries.
5. **Environment** — during live episode D: PL die temperature flat at
   44.2–44.9 °C across the onset (and *lower* than during clean periods the
   previous day); every SYSMON rail nominal and stable, including VCCINT at
   0.721 V ±2 mV — the correct nominal for the K26's -2LV speed grade, a
   value that briefly looked alarming until the speed file was checked.
6. **Configuration/software state** — `configuration_generation` constant
   from boot through every episode; live settings refreshes hours away from
   onsets; nothing in the journal at any onset or exit.
7. **The transport RTL's functional behavior** — the decisive elimination.
   A new exhaustive bench (`record_transport_phase_tb`, MSAP1_PL `ffac3eb`)
   instantiates the full chain exactly as `meter_core` wires it (result hub
   + aggregate producer → arbiter → packetizer) and emits a basic/aggregate
   pair at **every relative offset from −3 to +67 cycles under three AXIS
   backpressure regimes** (always-ready, 50 % duty, LFSR) — 213 alignment
   pairs, 426 records — with a scoreboard requiring every record delivered
   **exactly once**, every packet exactly 64 beats, and zero drops.
   **PASS at every alignment.** The RTL cannot duplicate or lose a record
   functionally.

## 5. What the forensics actually captured

Three record-level signatures, all at the stuck slot, all mutually
consistent with exactly one mechanism:

**(a) Wholesale substitution — the common case** (episode D, hundreds of
instances): the slot-9 basic record never arrives; in its place, that
window's MTR2 aggregate is delivered **twice, bit-perfect**
(`expected 4818, received 4817` — every duplicated field validating against
live configuration: magic, format, size 256, generation, sample rate,
15-block/180-cycle shape, basic-sequence coverage, sample index). The two
anomalies sit ~1.4 s apart (the lost basic mid-window, the duplicate at
window close), and the per-window record count is **conserved** at 16.

**(b) Hybrid records — the rare case** (episode C ×2, episode D ×1, and
retroactively the 2026-08-13 17:18:03 event): a record with the **basic
producer's head and the aggregate's zeroed tail** — valid MTR1 magic,
format, size, sequence (slot 9), generation, rate; zero first-sample index
(words 60/61) and zero nominal frequency (word 15) — exactly the words
where the MTR2 layout holds zeros. Episode D's instance was captured with
the complete word dump in the journal (`08-15 16:54:46`).

**(c) The exit burst**: the final event of every episode loses the slots
from the stuck slot through slot 1 — counter deltas show most of those
records never arrive at all (7 of 8 in episodes A/D), again with zero
drops anywhere.

Signature (a) rules out data corruption (nothing is corrupted — the wrong
*record* is captured); signature (b) rules out anything record-granular
(the boundary falls *inside* a record, at a bus-geography position); the
count conservation in (a) rules out drop/flow-control mechanisms (nothing
is dropped — one slot is read twice while another is skipped). Together
they describe a **capture-boundary race**: at the moment the packetizer's
2,048-bit capture registers sample the arbiter's combinational mux, some or
all bits resolve to the *other* producer's value.

## 6. The structural asymmetry — why MTR1 suffered and MTR2 never did

The as-built arbiter (`measurement_record_arbiter.vhd`, 46 lines) was
purely combinational:

```vhdl
m_valid_o  <= basic_valid_i or aggregate_valid_i;
m_record_o <= basic_record_i when basic_valid_i = '1'
              else aggregate_record_i;
```

The mux's **resting state is the aggregate record** (the `else` branch).
An MTR2 capture therefore involves *no bus transition at all* — the bus
already carries the aggregate whenever nothing else is happening —
structurally immune. An MTR1 capture requires all 2,048 bits to swing from
the aggregate's contents to the basic record's within one cycle against
the capture edge. Every observed asymmetry follows: only basics are ever
lost; the substituted-in record is always the aggregate; the hybrid
records' zeroed words are exactly the aggregate layout's zero region.

The producers themselves were exonerated by inspection and simulation: the
result hub assembles its entire record in a single clocked process
(atomic), the aggregate producer likewise, and the packetizer captures the
full 2,048-bit bus in one clock. No stage *between* flops can produce a
partial record — only the flop *boundary* can, if the bus is still moving
when the edge samples it.

### On the timing evidence — an honest correction

The routed checkpoint of the deployed bitstream showed the packetizer
capture bus with 4.6 ns of setup slack but only **0.016–0.024 ns of hold
slack** — initially reported in this investigation as a smoking gun. That
framing was **overstated and is corrected here**: re-extracting the same
paths on the *fixed* build shows similarly barely-met hold values, because
routers fix hold everywhere to barely-positive by construction; thin hold
slack alone does not distinguish a healthy path from the guilty one, and
sign-off STA formally covered these paths at corners. The conviction of
this fault class therefore rests on the elimination chain (§4) and the
capture signatures (§5) — everything except a physical race at this
boundary is excluded by measurement — not on the slack numbers. What the
timing data does establish is that this boundary was the *thinnest* margin
in the record path's neighborhood and the only place the observed
signatures can arise. The specific electrical mechanism (local coupling,
clock-network variation, aging, an OCV corner-model gap on this die) is
deliberately left open; the fix does not depend on which it is.

## 7. The fix

`measurement_record_arbiter` gains a **registered valid/ready output
stage** (MSAP1_PL `85ee806`): the arbitration decision loads a 2,048-bit
output register, and the packetizer captures from that register a full
cycle later — flop-to-flop, ~10 ns of margin instead of a same-edge race.
Producer handshakes and the Basic-first fixed priority are unchanged; the
entity gains `aclk`/`aresetn`, wired at its two instantiation sites
(`meter_core.vhd`, the phase-sweep bench). Cost: one clock cycle of added
latency against a 200 ms record cadence, and ~2 k flip-flops.

Whatever the precise electrical mechanism was, a registered handoff removes
the entire class: there is no longer any cycle in which the capture bus is
combinationally dependent on producer state.

**Verification (simulation):** `record_transport_phase_tb` (213 alignment
pairs × 3 backpressure regimes, exactly-once scoreboard) — PASS; original
`meter_packet_tb` wire-format regression — PASS unchanged; full metering
pipeline suite including grid timing and the HLS-integrated aggregator
tests — PASS. The bench is registered in CI
(`check_metering_pipeline.tcl`) as a permanent regression gate.

**Verification (implementation):** full non-incremental rebuild
(synthesis → implementation → bitstream) — the previous routing was
deliberately discarded rather than reused. Zero timing violations,
WNS 0.513 ns, WHS 0.010 ns (design-typical), bitstream written.

## 8. Reattribution of the 2026-08-13 incident

FIXRP00001 attributed the Aug-13 record loss to SD-card latency stalling
the blocking publish path until the (then 4-deep) kernel ring overran. That
mechanism was real, demonstrated end-to-end in code, and the fixes it
motivated (memory spool, WAL checkpoint repair, ring depth, overrun
attribution) are all independently justified and remain correct. But this
investigation established that the Aug-13 gap pattern — fixed slot ≡13,
one per window, exit burst of 4 = slots 13→1, the 17:18:03 hybrid record —
matches the transport-race signature exactly, on the *previous* bitstream,
and the discriminating counters (kernel overruns, `invalid_records`
sampling) did not exist then. The honest reading: **the Aug-13 record loss
was most likely this same PL fault**, with the storage chain a plausible
co-stressor at most. The fault therefore predates the 2026-08-14 bitstream
and is not a product of the new DCP build flow or the repackaged HLS IP —
consistent with a marginal boundary that different placements express at
different slots (13 vs 9).

## 9. What remains open, and the deciding experiment

**The one unexonerated suspect:** the HLS aggregator shim's *event* path,
upstream of the producers. The phase-sweep bench drives the producers
directly and therefore cannot exonerate the shim; a re-fired aggregate
event would also yield bit-perfect duplicates (though it would not, by any
identified mechanism, explain the lost basics or the hybrids). The
transport redesign proposal
(`MSAP1_PL/doc/future_plan/measurement_record_transport_redesign.md`)
shares exactly this blind spot — no transport change can fix an upstream
event fault — so the fix and the redesign stand or fall together on this
question.

**The deciding experiment — the 72-hour soak.** Episodes recurred 2–4×/day
across every observed boot, and the Gen-3 forensics detects an episode
within seconds. On the fixed bitstream:

- **Silence for 72 h** (zero `meter_sequence_gap`, zero
  `meter_record_stale_rejected` / `_config_rejected` / `_decode_rejected`,
  `invalid_records` flat, margin-watch clean): the arbiter race is
  confirmed as root cause; close this report, merge `fix/periodic_seq_gap`
  across the three repos, and schedule the redesign on its own (scaling)
  merits.
- **Any recurrence**: the transport is eliminated *in hardware*, and the
  investigation moves to the shim/event path with full forensics already
  deployed — the duplicate's word dump plus the shim's beat framing
  (`HLS_AGG_*` registers 0x90–0x98) are the first evidence to pull.

Soak checklist (device 172.30.19.99):

```
journalctl -f MNC_EVENT=meter_sequence_gap \
  + MNC_EVENT=meter_record_stale_rejected \
  + MNC_EVENT=meter_record_config_rejected \
  + MNC_EVENT=meter_record_decode_rejected
mnc meter health --output json   # invalid_records must stay flat
cat /tmp/margin-watch.csv        # sampler continues through the soak
```

Do not reboot the device after any suspected event before collecting the
journal (volatile storage).

## 10. Collateral improvements shipped during this investigation

- Complete rejection observability in the acquisition daemon: every path
  that discards a record now logs cause + raw words, rate-limited with
  suppression accounting (APU `236e1c6`, `0cae2ca`).
- Kernel-vs-PL loss attribution on every gap event (with FIXRP00001).
- `record_transport_phase_tb` exhaustive transport regression in CI
  (MSAP1_PL `ffac3eb`).
- Read-only margin-watch sampler pattern for environmental correlation
  (deployable in minutes, no software changes).
- Incident log bundle with full journals, histograms, and analysis:
  `MSAP1_DOC/raw_doc/incident_logs/2026-08-14/`.
- Design documents produced: record-transport redesign (per-producer
  FIFOs, packet-boundary arbitration) and RPU ADC health-poll hardening —
  both deliberately out of scope here, both informed by this incident's
  evidence.

## 11. Lessons

1. **Attribution beats inference.** The single most valuable change of the
   week was making every gap event state *where* the record was lost. The
   Aug-13 misattribution happened because two mechanisms produced identical
   logs; one counter field ended that ambiguity permanently.
2. **Instrument every silent path, not the one you suspect.** The first
   forensics generation covered the theorized rejection site and caught
   nothing — the fault used the *other* silent path. Coverage of all paths,
   with suppression accounting, is what made episode D decisive.
3. **Bit-perfect evidence kills theories fast.** "Corrupted record"
   survived until one duplicated record arrived with every field valid;
   the theory died in a single log line. Dump raw bytes, not summaries.
4. **Functional exoneration by exhaustive simulation is cheap and
   decisive.** 213 alignment pairs took minutes to run and converted "the
   RTL is probably fine" into an elimination strong enough to convict the
   physical layer by exclusion.
5. **State confidence honestly, and downgrade publicly.** Two theories in
   this report were retired by their own follow-up experiments (config
   mismatch; the 16 ps "smoking gun"). Writing the corrections into the
   record is what keeps the surviving conclusion trustworthy.
6. **Wide combinational buses into capture registers are a design smell**
   regardless of STA sign-off. The redesign's rule — serialize before you
   arbitrate, register every handoff — makes the whole class unrepresentable.

## 12. Artifact index

| Repo / location | Branch | Commits | Content |
|---|---|---|---|
| `applications/MSAP1_PL` | `fix/periodic_seq_gap` | `ffac3eb`, `85ee806` | Phase-sweep bench + registered arbiter fix; bitstream built |
| `applications/MSAP1_APU` | `fix/periodic_seq_gap` | `236e1c6`, `0cae2ca` | Rejection forensics generations 2–3 |
| `yocto-build/sources/meta-monutchee` | `fix/periodic_seq_gap` | `beff99b`, `be504ce` | Kernel ring 4→8, recipe hygiene (from FIXRP00001 scope) |
| `MSAP1_DOC/raw_doc/incident_logs/2026-08-14/` | — | — | Full journals, per-minute histograms, device snapshots, ANALYSIS.md |
| `MSAP1_PL/doc/future_plan/measurement_record_transport_redesign.md` | — | — | Long-term transport architecture |
| `MSAP1_RPU/docs/future_plan/ADC_HEALTH_POLL_HARDENING.md` | — | — | Separate ADC SPI health-poll issue |
| Device `/tmp/margin-watch.csv` | — | — | 1-min environmental/counter trace (running) |

---

# Part II — The fix that failed, the rewrite that exonerated the PL, and the kernel bug underneath

| | |
|---|---|
| Continuation window | 2026-08-15 17:31 → 2026-08-16 (open) |
| Outcome of Part I's fix | **Falsified.** Episode E recurred on the fixed bitstream, deployment verified by md5 chain |
| Intervening milestone | Complete HLS rewrite of the PL record path (removes the one suspect Part I §9 could not exonerate) |
| Decisive observation | The fault survived a total replacement of the PL datapath, signature bit-identical |
| Root cause (current) | Kernel cyclic-DMA period accounting: the Xilinx AXI DMA cyclic callback is **not** one-per-period, and every safety bound derived from the counter it fed |
| Fix | Completion markers in the DMA ring — availability observed, not inferred (`meta-monutchee` `31e802f`, `308a2f2`) |
| Status | Fix cross-builds clean (aarch64, `W=1`); **not yet proven on hardware** — see §20 |

## 13. Why there is a Part II, and the milestone timeline

Part I ended with one experiment outstanding and both outcomes pre-committed:
silence for 72 h would confirm the arbiter race, any recurrence would
eliminate the transport in hardware and redirect to the shim/event path.
The experiment returned **recurrence**. What follows is the chain that
opened, written as it happened, because the sequence is the argument: each
milestone eliminated a layer, and the layer that finally failed was the one
nobody had touched in three days of work.

| # | When (device UTC) | Milestone | Outcome |
|---|---|---|---|
| 1 | 08-15 17:31 | Part I fix `85ee806` deployed, 72 h soak armed | — |
| 2 | 08-16 00:54:55 | **Episode E** on the fixed bitstream | Arbiter race falsified as *sole* cause |
| 3 | 08-16 (day) | PL record path rewritten end-to-end in HLS | Shim/event path — Part I's last suspect — deleted, not patched |
| 4 | 08-16 05:57 | New bitstream + APU release deployed | Clean start |
| 5 | 08-16 06:07 | First verification: all counters zero, block accounting exact | Looked fixed |
| 6 | 08-16 09:28:52 | Gaps return on the rewritten datapath | **PL exonerated by replacement** |
| 7 | 08-16 10:44 | Signature characterised: phase-locked, 1/window, PL counters zero | Suspicion moves to the untouched layer |
| 8 | 08-16 (day) | Root cause identified in `msap1_dma_core.c`; fix built | Awaiting deploy |

## 14. Milestone 1–2 — the deciding experiment returned "recurrence"

Episode E began **2026-08-16 00:54:55** on the `85ee806` bitstream, with the
deployment verified end to end by md5 chain (`fa3450dc` `.bit` → SDT export
→ `a58b7ddd` firmware `.bin` on target), so "the fix was not actually
running" was excluded before anything else. The signature was unchanged
from episodes A–D: one basic record lost per aggregation window, that
window's aggregate delivered twice bit-perfect, every drop counter zero.

Part I's §9 branch therefore resolved: **the transport RTL is eliminated in
hardware, not merely in simulation.** The registered arbiter remains correct
and stays — it removed a genuine design smell and a real fault class — but
it was not the cause of the observed loss.

Live registers during episode E added a second elimination that mattered
more than it appeared at the time: the aggregation engine's own beat
counters (`AGG_RECORD_COUNT` 0x7C and its mirror 0x90, both carried *inside*
the aggregate beat) advanced **exactly once per window** while the APU
received that aggregate **twice**. Whatever duplicated the record did so
*after* the engine counted its output and *before* the APU decoded it.

## 15. Milestone 3 — the PL record path was rewritten, and the fault ignored it

The transport redesign (`measurement_record_transport_redesign.md`) was
already scheduled on scaling merits. Episode E promoted it: Part I §9 named
the HLS aggregator shim's level→event conversion as the last unexonerated
suspect, and the redesign *deletes that pattern* rather than patching it.

Shipped on branch `feat/hls_mtr1` (MSAP1_PL `3817dce`, `b9b3588`):

- MTR1 numerics, record construction and serialization collapsed into one
  Vitis HLS engine (`Mtr1Engine`), replacing `meter_rms` + `MeterResultHub`.
- The aggregation engine extended to consume the MTR1 engine's result
  stream directly and emit its own MTR2 records (`CycleAggregator`, since
  renamed `Mtr2Engine`).
- Retired to git history: `meter_rms`, `MeterResultHub_Wrapper`,
  `measurement_record_arbiter` (including Part I's fix), `MeterPacketizer_Wrapper`,
  `aggregate_record_producer`, and **`meter_cycle_aggregator_hls_shim`** —
  the suspect itself.
- Per-producer 32-bit AXIS streams into packet-mode `axis_data_fifo`s and an
  arbitrate-on-TLAST `axis_switch`, so nothing wider than 32 bits is ever
  arbitrated and no level→event conversion exists anywhere on the path.
- New in-fabric observability: `record_word_tap` republishes each producer's
  in-record health counters and watchdogs the 64-beat framing invariant.

Two defects were caught during the rewrite that are worth recording because
both are invisible to the tools that would normally catch them:

1. A two-process DATAFLOW form of the MTR1 engine **deadlocked C/RTL
   co-simulation** at the drop-stress case (at snapshot depths 1 and 2).
   Restructured to the single-shot pattern the aggregator already had
   hardware hours on — which also improved the semantics (every closed
   window is finalized; the RTL's calc-busy drop has no successor).
2. A **valid/ready deadly embrace** between the two engines: a raw HLS AXIS
   master gates TVALID on TREADY (AXI-illegal), and wired directly to the
   aggregator's TVALID-gated TREADY it deadlocked the chain. **Cosim cannot
   see this** — it inserts its own well-behaved FIFOs — and the retired VHDL
   shim had masked it by being the compliant middleman. Only the new
   whole-chain bench (`meter_record_stream_tb`, real shims + both packaged
   engines + both exported streams under TREADY backpressure) caught it.
   Both are now design rules in `MSAP1_PL/AGENTS.md`.

The rewrite deployed 08-16 05:57 with the matching APU release (v3/v2 record
formats). First verification at 06:07 was clean on every axis: zero
rejections, all drop counters zero, health PASS, both tiers decoding, and
the block-accounting identity reconciling exactly
(`RESULT_SEQ 2305 = 153×15 + 1 ineligible + 9 in flight`).

**At 09:28:52, after 3 h 21 m clean, the gaps returned.**

## 16. Milestone 4 — the signature that reframed the investigation

The recurrence was characterised over the following 75 minutes from a
tmpfs-resident sampler (deliberately no SD writes, so the instrument could
not perturb the publish-path latency under test):

| Evidence | Value |
|---|---|
| Rejection cadence | 100 per 5 min = **1 per 3 s**, i.e. exactly the aggregate rate |
| Lost basic sequences | 86919, 86934, 86949, 86964, 86979 — **exactly 15 apart** |
| Phase | all ≡ 9 (mod 15); the first loss at 09:28 (63459) has the **same residue** |
| Phase stability | held across 1500+ windows / 76 min (a free-running software timer would have drifted several blocks) |
| Conservation | in one 5-min sample the PL emitted 1503 basic + 100 aggregate = 1603 records; **exactly 100 basics were lost** — one per window |
| PL counters | result drops, emit drops, aggregate drops, shim-FIFO drops, aggregate resets, continuity errors: **all zero** |
| Aggregate self-report | each MTR2 record declares a **contiguous** folded range (63452..63466, then 63467..63481) — *including the basic records the APU never received* |

That last row is the one that localises the fault. The aggregation engine
demonstrably consumed every basic result; the records were emitted; the APU
never saw them. Combined with the PL's zeroed counters and the
backpressure-safe FIFO/switch path (which can stall but cannot drop), the
loss is downstream of the PL's AXIS boundary.

And the reframing question wrote itself: **the signature is bit-identical
across two completely different PL implementations. Look at what was not
replaced.** The kernel DMA driver had been in the design unchanged
throughout — and Part I had eliminated it (§4.1) on the strength of a
counter that, it turns out, could not have reported the fault.

## 17. Root cause — the cyclic callback is not a period counter

`msap1_dma_core.c` advanced its absolute completed-period counter by one per
cyclic completion callback:

```c
static void msap1_dma_period_complete(void *parameter)
{
    if (atomic64_cmpxchg(&mdev->produced, 0, ring_periods) != 0)
        atomic64_inc(&mdev->produced);       /* +1 per CALLBACK */
    wake_up_interruptible(&mdev->wait);
}
```

Its comment asserted the callback "runs on every completed period". It does
not. Verified in the vendor source for the exact kernel in use
(`drivers/dma/xilinx/xilinx_dma.c`), at all three layers:

- **Client callback:** `xilinx_dma_chan_desc_cleanup()` handles a cyclic
  descriptor with `xilinx_dma_chan_handle_cyclic()` followed by an immediate
  **`break`** — at most **one callback per cleanup call** — and never
  `list_del`s the cyclic descriptor.
- **Tasklet:** cleanup runs from a tasklet, and `tasklet_schedule()` is
  idempotent; N interrupts arriving before it runs produce one run.
- **Hardware:** the AXI DMA's IOC is a **latched status bit in `DMASR`, not
  a counter**. The ISR reads `DMASR` and acks the whole mask in one write, so
  two BDs completing before that ack are indistinguishable from one.

The chain is therefore `callbacks ≤ tasklet runs ≤ IRQs ≤ completed periods`
— it can only lose counts. Residue offers no substitute:
`xilinx_dma_get_residue()` sums `(control − status)` over the descriptor
segments and **nothing resets a segment's status between laps** of a cyclic
transfer, so residue collapses to zero after the first lap.

**Why the meter stream and no other:** the only closely spaced pair in the
entire measurement stream is the 15th basic record and the aggregate that
folds it, emitted microseconds apart once per aggregation window. Everywhere
else records are 200 ms apart and the tasklet always keeps up. So the
counter fell exactly one behind **per aggregation window**, structurally,
forever — which is precisely the observed cadence and the observed phase
lock.

**Why it was invisible:** `oldest_safe`, `msap1_dma_catch_up()` and
`overrun_periods` are all derived from that same counter. Under-counting
moved the reader's window and its loss detector *together*, so `read()`
handed out a ring slot the DMA had already overwritten while
`overrun_blocks` stayed at zero — and the APU's gap events faithfully
reported "no kernel transport overrun".

**Why it produced a bit-perfect duplicate rather than corruption:** nothing
is corrupted; a whole, valid, previously-written slot is read a second time.

## 18. The fix — observe the ring, do not infer from callbacks

`meta-monutchee` `31e802f`. The consumer stamps a 16-byte marker into the
tail of every period it takes, and treats a period as complete only once
that marker is gone. The DMA fills a period in ascending address order, so
the tail lands last: a partially written period still holds its marker and
is correctly withheld. Completion callbacks are demoted to wake-ups.

The change deliberately **removes the dependency on callback counting**
rather than compensating for a measured deficit — so it is correct whether
the culprit was callback coalescing, the first-callback `ring_periods` jump,
the ring index arithmetic, or the safe-window math, all of which routed
through the same counter.

Two previously silent failures become loud:

- a completely fresh ring proves the consumer was lapped, and is counted;
- a short packet (TLAST before the period fills — a PL framing fault) leaves
  its marker forever, so it is detected by a later period completing,
  counted, and stepped over instead of stalling the stream. This hazard was
  introduced by the marker scheme itself and closed in the same change.

`308a2f2` additionally exposes the raw callback count through the
always-zero `reserved` word of the transport-status ioctl (no size or offset
change; both APU mirrors updated in `3b24650`). This makes the diagnosis
falsifiable from a running system: **`produced_blocks − callbacks` is the
coalescing deficit**, and it should grow by ~1 per aggregation window.

## 19. Corrections to Part I

Recorded explicitly, in keeping with Part I §6's practice:

1. **§4.1 — "Kernel DMA transport eliminated" was wrong, and wrong for an
   instructive reason.** The elimination rested on `overrun_blocks` reading
   zero. That counter is computed from the same `produced` value the fault
   corrupts. *An instrument that shares state with the suspect cannot
   exonerate it.*
2. **§4.1 — the stale-slot refutation used the wrong frame.** Part I
   rejected ring replay because "a stale slot would replay content from
   exactly `ring_periods` (8) records earlier, but the duplicate arrived
   immediately after its original". With **16 records per aggregation
   window** (15 basic + 1 aggregate) and an **8-slot ring**, 16 ≡ 0 (mod 8):
   every aggregate lands in the *same* ring slot every window. A re-read of
   that slot therefore returns the **previous aggregate** — ring-distance 8
   in periods, but *adjacent* in aggregate-sequence terms. The observation
   was correct; the inference from it was not.
3. **§9 — the shim/event path is now exonerated too**, by deletion: it no
   longer exists, and the fault persisted unchanged.
4. **The Aug-13 reattribution (§8) partially reverts.** FIXRP00001's
   original instinct — a kernel-ring accounting problem — was closer to
   correct than Part I's PL attribution. The storage/fsync chain remains a
   plausible *co-stressor* (tasklet latency raises the odds of two
   completions landing in one interrupt), which also explains the episodic,
   self-clearing character that a purely structural bug would not.
5. **Part I's fix is not withdrawn.** The unregistered 2,048-bit
   combinational handoff was a genuine defect; it is simply not what was
   losing records. It has since been deleted along with the rest of the old
   transport.

## 20. Status, and the deciding experiment (again)

**This root cause is not yet proven on hardware.** It is a static
verification (vendor source + hardware semantics) plus a mechanism that
explains every observation — cadence, phase lock, conservation, zeroed PL
counters, bit-perfect duplication, survival across two PL implementations,
and the episodic onset. That is strong, and it is not the same as measured.

After deploying the kernel module:

- **Confirmation:** `gaps` and `invalid` stop climbing, **and**
  `produced_blocks − callbacks` grows at ~20/min (one per aggregation
  window, tracking the MTR2 rate).
- **Falsification:** gaps persist **and** `produced_blocks − callbacks ≈ 0`.
  The coalescing mechanism is then wrong; next suspects are PL-side packet
  counting (see below), ring index/phase arithmetic, and cache/coherency
  assumptions.
- **Expect `overrun_blocks` to become non-zero.** That is the fix working,
  not a regression: it is loss that was previously silent. Sustained growth
  points at consumer stalls, where the historian WAL checkpoint defect
  (FIXRP00001) remains a standing suspect.

**Known observability gap, introduced by the rewrite:** `record_word_tap`
sits exactly on the `TVALID && TREADY && TLAST` boundary and already detects
framing violations, but it has no packet counter and its `framing_error_o`
is wired to `open` in `meter_core`. The PL therefore cannot currently state
"I transmitted N packets". Closing this needs a bitstream rebuild, so it
should not gate the kernel deploy — but it is the measurement that would
have made this diagnosis a five-minute arithmetic check rather than a source
audit.

## 21. Lessons (Part II)

1. **An instrument that shares state with the suspect cannot exonerate it.**
   The kernel was cleared for three days by a counter the fault corrupted.
   When an elimination rests on a single counter, ask what that counter is
   derived from.
2. **Replacing a subsystem is a legitimate experiment.** The HLS rewrite was
   not undertaken as a diagnostic, but by replacing essentially all of the
   PL record path and leaving the signature bit-identical, it eliminated
   more suspects in one deployment than three days of instrumentation had.
   When a fault survives a rewrite, the answer is in what you did *not*
   rewrite.
3. **Phase-lock is a fingerprint.** "One per aggregation window" was
   suggestive; "exactly 15 apart with the same residue across 1500 windows"
   was decisive — it ruled out every free-running software timer and pointed
   at something structurally tied to the aggregation cadence.
4. **Verify vendor semantics in the vendor's source, for the exact kernel.**
   The DMAengine cyclic contract is *period-oriented in intent*; this
   implementation does not honour it per-period. A reviewer's challenge to
   an asserted (rather than verified) claim is what forced this to be
   settled from source — and it was right to insist.
5. **Prefer fixes that remove a dependency over fixes that correct it.**
   Compensating `produced` for a measured deficit would have been a patch
   against one mechanism. Observing the ring removes the whole class of
   accounting faults, and stays correct even if the specific mechanism is
   later disproven.
6. **A fix can be correct and still not be the cure.** Part I's arbiter fix
   was good engineering on a real defect, and it moved nothing. Ship it,
   record it, and keep the experiment that would falsify it.

## 22. Artifact index (Part II)

| Repo / location | Branch | Commits | Content |
|---|---|---|---|
| `yocto-build/sources/meta-monutchee` | `feat/hls_mtr1` | `31e802f`, `308a2f2` | **The fix**: ring completion markers; callback-count diagnostic in the status ioctl |
| `applications/MSAP1_APU` | `feat/hls_mtr1` | `e0991f9`, `3b24650` | v3/v2 record decode; transport-status mirror update |
| `applications/MSAP1_PL` | `feat/hls_mtr1` | `3817dce`, `b9b3588` | HLS record-path rewrite; `Mtr2Engine` rename + shim symmetry |
| `applications/MSAP1_WEB` | `feat/hls_mtr1` | `26ae17a` | Readings field renames |
| `MSAP1_PL/.../tb/meter_record_stream_tb.sv` | — | — | Whole-chain bench that caught the AXIS deadly embrace |
| `MSAP1_PL/doc/future_plan/measurement_record_transport_redesign.md` | — | — | Rewrite plan, with the episode-E gate recorded in §3 |
| Device `/tmp/meter_soak.log` | — | — | tmpfs sampler: 5-min registers + APU counters (volatile; collect before reboot) |
