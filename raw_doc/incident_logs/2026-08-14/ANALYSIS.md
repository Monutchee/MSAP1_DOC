# PL record-emission fault — episode log capture, 2026-08-14

Device msap1, boot 13:53:29 UTC, bitstream md5 09c64e18… (MSAP1_PL
d83e97d, first hardware run of the feat/enable_dcp flow).

## Files

- `full-journal-boot-20260814.log` — complete journal for this boot (2,944 lines).
- `sequence-gap-events-verbose.log` — every `meter_sequence_gap` event with
  structured fields (`MNC_TRANSPORT_OVERRUNS`, `MNC_MISSING_RECORDS`, …).
- `gap-histogram-by-minute.txt` — events per minute (episode boundaries).
- `device-state-snapshot.txt` — health counters + PL registers at 23:11:15 UTC.

## Episodes

| | Episode A | Episode B |
|---|---|---|
| Onset (UTC) | 17:17:08 | 21:19:17 |
| End (UTC) | 17:50:56 | 21:25:50 |
| Duration | 33 min 48 s | 6 min 33 s |
| Gap events | 677 | 132 |
| Stuck slot (seq mod 15) | **9** | **9** |
| Steady-state loss | 1 rejected record/window, silent (`matches_configuration`) | same |
| Exit burst | 8 missing (seq 71229→71237) | 8 missing (seq 135699→135707) |
| Kernel overruns | 0 throughout | 0 throughout |

## Hard structural facts

1. **Same stuck slot both episodes**: the rejected record is always wire
   sequence ≡ 9 (mod 15) — mid-window (window close is at slot ≡ 0–1).
2. **Exit-burst arithmetic**: both exits lost exactly the slots from the
   stuck slot through slot 1 of the next window (9,10,…,14,0,1 = 8 records).
   The 2026-08-13 incident on the PREVIOUS bitstream ended the same way:
   stuck slot 13 → exit burst 13,14,0,1 = 4 records. Three episodes, one
   formula: `exit_burst = (1 − slot) mod 15 + 1`. The exit event drops the
   corruption zone through the window boundary once, then the pipeline runs
   clean.
3. **Exit records mostly vanish, not reject**: counter deltas show ~7 of
   each 8-record exit burst never arrived at the APU at all (1 rejected),
   with kernel overruns 0 and PL drop counters (0x88/0x8C/0x98) all zero —
   the loss stage has no counter.
4. **Onset spacing is quasi-periodic ~3.4–3.5 h**: boot→A = 12,219 s,
   A-end→B-start = 12,501 s. Two intervals are thin evidence, but if it
   holds, the next onset falls around **00:45–00:50 UTC (2026-08-15)**.
   No journal/config/grid/thermal trigger at either onset.
5. PL block accounting (RESULT_SEQ = AGG×15 + ineligible + in-flight)
   reconciled exactly at every check, including across both exits;
   aggregation never reset; aggregates flowed at 20/min throughout.

## Episode C update (2026-08-15, instrumented boot)

09:04:10–10:55:03 UTC, 111 min, stuck slot 9 again, exit burst 8 again
(199374→199382 = slots 9→1 — the formula's third confirmation). Decisive
new evidence from the first instrumented episode:

- `invalid_records` tracked the ~2,222 gaps 1:1 but **zero**
  `meter_record_config_rejected` entries fired → the swallow site is the
  **stale/out-of-order half-range guard**, not `matches_configuration()`.
  The prior "config mismatch" attribution over-read the counter lockstep,
  which never discriminated between the two silent paths.
- Twice, the decoder caught the record instead (seq 196389 and 197574 —
  both slot 9): **valid words 0–5, zeroed words 60/61
  (first_sample_index = 0), invalid word-15 nominal frequency** — with
  matching `meter_sample_range_gap` events ("expected 5027195953, got 0").
  Identical signature to the 2026-08-13 17:18:03 event.
- Unified mechanism: the slot-9 record is emitted **partially written —
  intact front, zeroed tail**. Truncation before word 3 → sequence reads
  0 → silently swallowed as stale (the common case); truncation mid-record
  → valid header, zeroed tail → decoder/sample-range events (the rare
  case). One fault, three faces.
- Onset was 9.2 h after boot: the ~3.4 h rhythm hypothesis is dead.
  Durations so far: 34 min, 6.5 min, 111 min.

Sharpened PL hypothesis: a **read-before-write race on the packetizer's
record buffer** (reader streams a slot before the writer finishes filling
it), latching into a fixed window phase for the episode's duration.
Forensics extended 2026-08-15 (`meter_record_stale_rejected` +
word dumps on the decoder path) to capture the truncation-point
distribution on the next episode.

## Episode D (2026-08-15 16:13:39–17:01:22, full forensics + live sampler)

The decisive capture, 48 min. Per window: the slot ≡9 basic never arrives
AND that window's aggregate arrives twice — **bit-perfect duplicates**
("expected 4818, received 4817", every field validating), the two anomalies
1.4 s apart. Record count conserved (16/window). Exit burst 8 = slots 9→1
(fourth confirmation). One hybrid caught with the full word dump at
16:54:46: MTR1 v2, valid words 0–5 (slot 9), zero first-sample index.
Environment excluded live: PL die 44.2–44.9 °C flat across onset, all
SYSMON rails nominal (VCCINT 0.721 V ±2 mV — the K26 -2LV nominal), kernel
ring stale-slot replay refuted (duplicate adjacent, not ring-distance 8).

Verification run the same day: the phase-sweep bench
(`record_transport_phase_tb`, 213 alignment pairs × 3 backpressure regimes)
proves the transport RTL **functionally incapable** of duplicates or
losses; the fix (registered arbiter output, MSAP1_PL `85ee806`) passes the
full simulation suite and builds clean (WNS 0.513 ns, zero violations).

Confidence framing, stated honestly: the earlier "16 ps hold slack =
smoking gun" reading is weakened — the rebuilt (registered) design shows
the same barely-met hold values on the capture bus, which is normal
router behavior (hold is fixed to barely-positive everywhere), so thin
hold slack alone does not distinguish the builds. What stands: functional
and environmental exoneration leave a physical/electrical race in the old
combinational 2048-bit handoff as the leading cause class, and the
registered handoff eliminates that class by construction. The one path
NOT yet exonerated is upstream of the producers: the HLS shim → aggregate
event plumbing (a re-fired aggregate event would also produce bit-perfect
duplicates), which the bench bypasses. **The 72 h soak on the fixed
bitstream is therefore the deciding experiment**: silence confirms the
arbiter race; recurrence redirects the investigation to the shim/event
path with the forensics already in place.

## Standing interpretation

Latched corrupt state in the PL record-emission path (result-hub output
register → combinational record arbiter → packetizer two-deep buffer),
entering a mode where one fixed pipeline slot per aggregation cycle is
emitted with corrupted header words, persisting minutes-to-tens-of-minutes,
and exiting by dropping the stuck slot through the window boundary. The
2026-08-13 incident (old bitstream) matches the same exit arithmetic, so the
defect predates the d83e97d bitstream and the original SD-latency/ring
attribution for 08-13 is likely wrong or at most a co-trigger.

Missing datum: WHICH header word is corrupt. Requires the rejection-logging
patch in `record_ingestor.cpp` (`matches_configuration` path) — deploy
before the predicted next onset to capture it.
