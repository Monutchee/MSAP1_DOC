# MSAP1 150/180-cycle Class A aggregation and the PL measurement record bus

## Document purpose

This document describes the implemented second IEC 61000-4-30 measurement
tier: the 150-cycle (50 Hz nominal) / 180-cycle (60 Hz nominal) fundamental
aggregate, together with the reusable PL measurement record bus introduced to
carry it. It is source material for a future developer guide.

The document covers:

- the measurement model, and the three things this tier deliberately is not;
- the PL aggregator: eligibility, interval integrity, and the arithmetic;
- the measurement record bus, its arbiter, and the MTR2 record format;
- the RPU's (deliberately minimal) role;
- APU decoding, validation, quality semantics, and exposure over IPC/REST;
- the Web measurement-interval selector and the corrected period labelling;
- the software reference aggregator and the full verification inventory;
- the two review passes that hardened the feature before merge;
- deployment coupling and explicit future work.

This tier builds directly on the timing foundation. Read first:

```text
applications/MSAP1_DOC/raw_doc/impl_report/DOCPR00009_CLASS_A_TIMING_FOUNDATION.md
```

The normative timing model and the per-module register/record contracts remain:

```text
applications/MSAP1_APU/docs/TIMING_MODEL.md
applications/MSAP1_PL/SourceData/DesignFile/MeterProcessing/README.md
applications/MSAP1_PL/SourceData/DesignFile/MeterCore/README.md
```

If a bit-level detail differs between this report and those contracts or the
source, the contracts and the source are authoritative.

Merged to `main` on 2026-08-09:

```text
MSAP1_PL   PR #19  merge 5eb9c0b   aggregator + measurement record bus
MSAP1_RPU  PR #17  merge f4eb5f8   aggregation health readback
MSAP1_APU  PR #33  merge c95bbca   MTR2 decode + IPC/REST exposure
MSAP1_WEB  PR #16  merge c0c3c62   measurement-interval selector + labelling
```

---

## 1. The measurement model

IEC 61000-4-30 builds its measurement hierarchy from the basic measurement
block upward. The basic block is cycle-defined — 10 cycles at a declared
50 Hz nominal, 12 at 60 Hz — and the next tier is formed from exactly **15
consecutive basic blocks**:

```text
50 Hz nominal:  15 x 10 cycles  ->  150-cycle aggregate
60 Hz nominal:  15 x 12 cycles  ->  180-cycle aggregate
```

Nominally about three seconds, but the duration is a consequence of the grid,
never a definition.

Three properties were treated as non-negotiable, because each one is a way the
tier could have been implemented incorrectly while still producing
plausible-looking numbers:

1. **It aggregates standardized Basic results, not raw samples.** The engine
   consumes the internal Basic measurement result event — the same event the
   Basic record producer consumes — so it is an aggregator, not a second
   independent RMS engine running its own window over the ADC stream. A
   second engine would drift from the Basic tier and would not be a Class A
   aggregate of Class A intervals.
2. **It closes on 15 blocks, never on a timer.** There is no wall-clock
   comparison anywhere in the close path. A three-second timer would produce
   intervals that silently vary in cycle content as the grid frequency moves.
3. **The 15 inputs must form one contiguous, homogeneous interval.** Skipping
   an ineligible block and quietly substituting a later one would change the
   measured interval while still emitting a record that claims to be 15
   blocks.

---

## 2. Architecture and ownership

```text
                          PL METROLOGY DOMAIN
ADC -> conversion -> RMS + frequency
                          |
                 Basic measurement result event
                          |
          +---------------+----------------+
          |                                |
   Basic record producer          150/180 aggregator
   (MeterResultHub, MTR1 v2)              |
          |                      Aggregate record producer (MTR2)
          |                                |
          +---------------+----------------+
                          |
              Measurement record bus + arbiter
                          |
                    Packetizer -> AXI DMA
                          |
                          v
                         APU
        MeterRecordStream (SQLite WAL)  +  MeterDecoderRegistry
                 |                              |
          basic decoder                  aggregate decoder
                 |                              |
                 +--------------+---------------+
                                |
                       MeterData / IPC / REST -> Web
```

Ownership is unchanged from the timing foundation and was a design constraint
of this milestone:

- **PL** owns cycle timing, the monotonic sample counter, and now the
  authoritative aggregation.
- **RPU** remains a configuration and control conduit plus health readback. It
  never consumes Basic results, never computes an aggregate, and aggregate
  measurement data never travels over RPMsg.
- **APU** owns decoding, durable storage, UTC mapping, and publication. It
  never recomputes the aggregate.
- **WEB** presents it.

RPMsg stays control-plane; DMA stays data-plane.

---

## 3. PL implementation

### 3.1 The internal Basic result event

`MeterCommon/measurement_record_bus_pkg.vhd` formalizes the event both
consumers read, so the aggregator never has to decode an MTR1 packet:

```vhdl
type basic_measurement_result_t is record
  valid, result_sequence, generation, sample_rate_hz, sample_count,
  valid_mask, status, rms_q16,          -- measurement
  first_sample, cycle_count, nominal_hz, flags,  -- closed-block provenance
  frequency_millihz, frequency_valid;
end record;
```

The provenance fields are the ones `grid_cycle_timing` latches atomically at
block close. Note `result_sequence`: `sequence` is a reserved word in
VHDL-2008, which is a trap worth recording for anyone extending this record.

`meter_core.vhd` composes the event from the RMS result, the grid-timing
provenance outputs, and the frequency estimate, and fans it out to both the
Basic record producer and the aggregator.

### 3.2 Eligibility and interval integrity

A Basic result joins an aggregate only if **all** of the following hold. The
first four mirror the APU's `class_a_aggregation_eligible()`; the rest are
interval-homogeneity rules:

| Rule | Violation counted as | Does the block seed the next aggregate? |
| --- | --- | --- |
| `cycle_locked`, not `free_run_fallback`, not `first_block_after_apply`, cycle count matches nominal | ineligible | **No** |
| Same configuration generation | reset | Yes |
| Same nominal frequency | reset | Yes |
| Same sample rate | reset | Yes |
| Consecutive `result_sequence` (modulo 2^32) | continuity + reset | Yes |
| Gapless `first_sample = previous first_sample + previous count` | continuity + reset | Yes |
| Configuration APPLY | reset | (tracking restarts) |

The seeding distinction is the important one. An **ineligible** block never
seeds, so a fallback or partial block cannot become the first member of the
next interval. A block that merely *differs* from the open aggregate
(generation, nominal, rate, or a discontinuity) discards the partial aggregate
and then seeds a fresh one, so the engine re-aligns on the next natural
boundary instead of waiting out a lost interval.

Sequence continuity and sample-range continuity are **both** enforced. They
catch different faults — a lost result event versus a sample-domain
discontinuity — so neither substitutes for the other. The sample-rate test is
defensive: the APU's configuration fingerprint already makes a rate change
alter the generation, but MTR2 reports one sample rate for the whole interval,
so the invariant is checked directly rather than inferred.

### 3.3 Aggregation arithmetic

Per quantity, because the standard does not define one rule for everything:

- **Voltage and current RMS** aggregate as the square root of the arithmetic
  mean of the squares of the 15 Basic RMS values, unweighted — each Basic
  interval contributes equally even though its sample count varies with grid
  frequency:

  ```text
  X_agg = floor( sqrt( floor( sum(X_i^2) / 15 ) ) )
  ```

  computed in the internal Q16 domain, so no precision is lost by passing
  through the public micro-unit format first.

- **Frequency** aggregates as the arithmetic mean `floor(sum(f_i)/15)`,
  published only when every contributing block reported a valid frequency.
  This is explicitly **informative**: the standardized Class A frequency
  product is defined over its own 10 s interval, which is not implemented.
  Section 6.3 describes how the APU prevents it from being mistaken for a
  measurement.

**Arithmetic geometry**, documented in the package so it can be audited
without reading the FSM:

| Stage | Width | Note |
| --- | --- | --- |
| Input | signed 64-bit Q16 | magnitude < 2^63 |
| Square | unsigned 126 bits | |
| Accumulator | unsigned **132 bits** | 15 x 2^126 < 2^130, two bits of margin |
| Mean | floor(acc / 15) | bit-serial division |
| Result | floor(sqrt(mean)) | 64-bit binary search |

Every rounding step is floor, and no stage can overflow by construction, so no
implicit HDL truncation or saturation participates in the result. That
property is what makes a bit-exact software reference possible (section 7).

**Timing.** The finalize chain is sequential: per channel one divide-prepare
clock, 132 divider iterations, and 64 square-root iterations of two clocks
each — about 261 clocks per channel. Across 7 channels plus the frequency
mean and emit, a complete finalize is roughly 2,000 clocks, about 20 µs at the
99.999 MHz AXI clock. Basic blocks arrive roughly every 200 ms, so the margin
is about four orders of magnitude. A Basic result arriving mid-finalize is
treated as a coherency loss and counted, rather than silently mixed in.

### 3.4 The measurement record bus

The DMA path was generalized from an MTR1-only producer into a record
transport. Producers publish complete fixed 256-byte records on a
valid/ready port; `measurement_record_arbiter` forwards them to the single
existing packetizer and AXI DMA channel with fixed priority, Basic first.

Two properties make this safe:

- **No backpressure into measurement.** Each producer keeps one newest-wins
  pending record and counts replacements, exactly like the original
  hub/packetizer pair. A stalled DMA drops and counts; it never stalls the
  metrology engines.
- **The arbiter cannot corrupt a record mid-transmission.** The packetizer
  hardwires `record_ready_o <= '1'` and latches the full 2048-bit record into
  its own registers in a single clock, then serializes 64 beats from that
  copy. The combinational mux is therefore sampled once per record and can
  never switch under an in-flight transmission. When Basic and aggregate are
  pending together, Basic is accepted first and the aggregate on the following
  clock; neither is discarded.

Starvation is not reachable at the product record rates (Basic ~5/s,
aggregate ~1 per 3 s) because a record drains in 64 beats. Future producers
(harmonics, PQ events) add an arbiter port, not a DMA channel.

### 3.5 MTR2 aggregate record

A new record type rather than an overloaded MTR1: word 0 keeps the container
magic `MTR1` that the DMA/stream layer keys on, and word 1 carries the format
the APU decoder registry dispatches on.

| Word | Content |
| ---: | --- |
| 0 | magic `MTR1` (`0x3152544D`) |
| 1 | format `0x00020001` |
| 2 | byte length = 256 |
| 3 | aggregate sequence (independent counter) |
| 4 | configuration generation |
| 5 | sample rate (Hz) |
| 6 | total sample count across the 15 blocks |
| 7 | valid mask, ANDed across contributors |
| 8 | status: bit 0 arithmetic error, bit 1 complete, bit 2 frequency valid |
| 9 / 10 | first / last contributing Basic sequence |
| 11 | [7:0] block count = 15, [15:8] nominal Hz, [31:16] total cycles |
| 12 / 13 | first sample index, low / high (64-bit measurement counter) |
| 16–31 | 8 channels x 2 words, aggregate RMS in signed 64-bit micro-units |
| 32 | aggregate frequency, millihertz (informative) |
| 33–63 | reserved, zero |

The last sample index is derived (`first + count - 1`), not stored. The first
sample index counts in the *same* 64-bit conversion-domain counter as MTR1 v2,
so aggregates and basic blocks share one measurement timebase and one UTC
mapping — but note it sits at **words 12/13 here and words 60/61 in MTR1 v2**.
The offsets differ; only the counter is shared.

Sequences are **independent per producer** — an aggregate stream numbered
from 1 alongside the Basic stream — with the relationship made explicit by the
first/last Basic sequence fields rather than by a shared counter.

### 3.6 Aggregation health registers

Read-only, in the RPU-owned processing block at base `0xB0050000`:

| Offset | Register | Meaning |
| --- | --- | --- |
| `0x78` | `AGG_STATUS` | [4:0] blocks in the open aggregate, [8] aggregate in progress |
| `0x7C` | `AGG_RECORD_COUNT` | completed aggregates |
| `0x80` | `AGG_RESET_COUNT` | partial aggregates discarded, any cause |
| `0x84` | `AGG_INELIGIBLE_COUNT` | Basic inputs rejected by eligibility |
| `0x88` | `AGG_CONTINUITY_COUNT` | sequence or sample-range discontinuities |
| `0x8C` | `AGG_DROP_COUNT` | aggregate records replaced before transport |

Counting per cause is deliberate: "aggregates are not appearing" is a very
different fault from "aggregates are appearing but the source blocks are
ineligible", and the counters separate them without a debug build.

---

## 4. RPU implementation

Minimal by design. The RPU reads the six aggregate health registers into its
internal meter status, following the existing grid-status pattern, and does
nothing else.

What the RPU explicitly does **not** do, and must not acquire:

- it does not consume Basic measurement results;
- it does not compute any aggregate;
- aggregate measurement data never travels over RPMsg — it stays on the meter
  DMA path.

RPMsg carries configuration, control, health, and acknowledgements only.

---

## 5. APU implementation

### 5.1 Types and decoding

`meter_timing.hpp` gains `AggregateTiming`, carrying the full provenance:
aggregate sequence, configuration generation, first sample index, total sample
count, first/last Basic sequence, block count, cycle count, nominal frequency,
arithmetic-error and frequency-valid status, time quality, and the optional
UTC start with its uncertainty.

`meter_record.hpp` gains the MTR2 format constant and typed accessors for
every word above. The decoder registry — already keyed on record word 1 —
gains an aggregate decoder producing
`MeasurementPeriod::Cycles150_180`. Basic decoding is untouched, and the v1
decoder remains registered so stored streams stay readable.

### 5.2 Defensive validation

The decoder validates the record's self-declared identity before building
anything, mirroring the hardened basic rules:

- the record must be marked **complete** — the producer sets that bit on every
  emitted aggregate, so a record claiming otherwise is corruption, not a
  partial to salvage;
- nominal frequency must be 50 or 60;
- block count must be exactly 15;
- cycle count must equal 15 x cycles-per-basic-block for that nominal;
- the first/last Basic sequence span must be exactly 15 consecutive blocks,
  computed modularly so a span wrapping `0xFFFFFFFF` is accepted;
- sample count must be non-zero;
- `first_sample_index + sample_count` must not overflow the 64-bit counter.

A rejection is counted as an invalid record at the ingest boundary. Decoding
happens **before** the WAL append, so a malformed record never enters durable
storage.

### 5.3 Ingest continuity

The DMA stream now interleaves two record formats with independent sequence
counters, so continuity tracking became per-format: basic records keep their
sequence and sample-range rules, aggregates get their own sequence baseline
and gap counter. Aggregates deliberately have no aggregate-to-aggregate
sample-range rule — the PL enforces continuity *within* an interval, and
consecutive aggregates may legitimately be separated by a reset.

`latest_record_`, which feeds `GET /api/v1/meter/readings`, remains
basic-only.

### 5.4 Quality semantics

Two rules matter more than they look:

- **An aggregation arithmetic error outranks the channel valid mask.** A
  saturated aggregate whose channel mask happens to be set would otherwise
  publish as `MeasurementQuality::valid` and hide the fault. Priority is
  therefore arithmetic error, then channel validity, then unavailable.
- **The `Cycles150_180` frequency is never advertised as valid.** Its Reading
  quality is always `unavailable`, and the REST payload carries
  `"informative": true` with deliberately no validity flag. The value and the
  producer's own validity flag remain available for diagnostics through
  `AggregateTiming::frequency_valid`. This is a presentation decision, not a
  change to the PL calculation: the standardized frequency product belongs to
  the 10 s interval and is not implemented.

### 5.5 Exposure

The newest aggregate record is cached beside the basic one in the ingestor,
carried in `InfoResponse`, and rendered by a new endpoint:

```text
GET /api/v1/meter/aggregate
```

It answers HTTP 200 whenever acquisition answers, returning
`{"available": false}` until the first aggregate exists. Needing about three
seconds of consecutive eligible blocks is a normal startup state, not a fault,
and modelling it as one would produce false alarms every restart.

The available payload carries the full provenance plus a channel array whose
naming, units and scaling are shared with `/meter/readings` through a common
helper, so the two documents cannot drift apart.

**Time-quality provenance.** The reported `time_quality` is the quality that
applied when the aggregate was **ingested**, not the daemon's state at request
time. An aggregate measured while synchronized but read back during holdover
would otherwise be mislabelled, and across a three-second interval that window
is real. Carrying it required an acquisition IPC version bump; the version is now 19.

---

## 6. Web implementation

A **Measurement interval** selector in the RMS readings heading chooses the
tier:

- *Basic block (10/12 cycles)* — the existing readings endpoint and poll,
  behaviour unchanged;
- *Aggregate (150/180 cycles)* — the new endpoint, polled once per second and
  only while selected, with the interval torn down on tier change and unmount.

Details that matter for correctness rather than appearance:

- aggregates arrive every ~3 s while polling is 1 s, so the sparkline extends
  only when the aggregate **sequence advances** — otherwise the trace would
  repeat flat points;
- history is discarded on tier change and on configuration-generation change,
  so a trace never spans two configurations;
- invalid channels serialize as `rms 0` and are excluded from both the trace
  and the min/max, which would otherwise be dragged to zero;
- the pending state explains that the first aggregate needs ~3 s of
  consecutive eligible blocks, rather than reporting a fault;
- the aggregate frequency is labelled informative, naming the 10 s interval.

**Period labelling was corrected in the same change.** Every "200 ms"
measurement label was removed, including the fallback strings for older
records: a basic block is cycle-defined and its duration tracks the grid, so a
fixed millisecond period is the wrong definition. Durations now appear only as
explicitly marked approximations. Genuine milliseconds — HTTP poll intervals,
record ages — were deliberately left alone. A block that closed on the
free-run fallback window is no longer labelled as cycle-defined either.

---

## 7. The software reference aggregator

The milestone required verifying the PL aggregate against an independent
implementation. `tests/support/reference_aggregator.hpp` computes the expected
aggregate from 15 Basic inputs using 128-bit accumulation and integer floor
square root — the same floor-everywhere arithmetic as the RTL, so it is
**bit-exact** rather than approximately equal, and a golden test can assert
equality instead of a tolerance.

It is explicitly non-authoritative and test-only: it lives under `tests/`, is
documented as a verification aid, and no production target can include it.
The PL remains the authoritative aggregator; the APU never recomputes.

---

## 8. Verification

**PL simulation.** `meter_cycle_aggregator_tb` embeds a wide-integer golden
model of the pinned arithmetic and drives twelve scenarios: constant inputs
(aggregate equals the input), deliberately varying inputs (so a mean-of-values
or wrong-divisor bug cannot pass), 50 Hz and 60 Hz shapes, off-nominal
sample-count variation, a fallback input, generation change, nominal change,
sample discontinuity, a Basic sequence gap with an intact sample chain,
a uint32 sequence wrap, a mid-aggregate sample-rate change, an APPLY reset,
maximum-magnitude inputs (overflow proof), and an invalid frequency input.

`meter_core_tb` adds an end-to-end scenario: 15 locked 10-cycle blocks driven
through the real AD7771 serial interface produce exactly one word-exact MTR2
record with clean counters. Module-reference inference and focused synthesis
of `MeterCore_Wrapper` both pass.

**APU tests.** `aggregate_decode_test` covers identity and varying golden
vectors, 50/60 Hz end-to-end decode of reference-built records, every
validation rejection, the RMS quality priority, and the informative-frequency
rule. `meter_aggregate_route_test` pins the serialized JSON byte-for-byte.
`acquisition_architecture_test` covers interleaved-stream continuity and the
aggregate cache. All host tests pass; the aarch64 cross build is clean.

**Hardware.** Validated on the production board at 128 kSPS with live grid
input: aggregate #378 built from Basic sequence 5657..5671 — exactly 15
blocks — 180 cycles at 60 Hz nominal, no arithmetic error, time synchronized,
age ≈ 2,951 ms matching the expected cadence. Independently, a sample of the
record stream showed 187 basic to 13 aggregate records, a 14.4:1 ratio
consistent with 15:1. Pipeline health was clean throughout: zero DMA, invalid,
gap, FIFO, and header errors.

---

## 9. Review passes

The feature was hardened twice before merge, and both passes are worth
recording because they show the failure modes this kind of code has.

**Timing-foundation hardening** (prerequisite, merged earlier) fixed
closed-block provenance atomicity in PL and, on the APU, bound UTC sync points
to their configuration, required a trusted external clock before reporting
`Synchronized`, propagated timestamp uncertainty, and added the
`class_a_aggregation_eligible()` predicate this tier mirrors.

**Aggregation review** produced the second PL commit and the MTR2 decoder
hardening. Findings, all confirmed against the code:

- the PL enforced generation, nominal and sample-range continuity but not
  Basic **sequence** consecutiveness, so a lost result event could pass
  unnoticed whenever the sample chain happened to stay continuous;
- sample-rate consistency was inferred from the generation rather than checked;
- the APU decoder ignored the aggregate arithmetic-error bit when setting RMS
  quality, while correctly honouring it for frequency — an inconsistency that
  could publish a saturated value as valid;
- the decoder did not require the `complete` bit;
- the informative frequency mean was being published as a valid measurement;
- the first/last Basic sequence span was copied but never validated.

A separate adversarial review of the Web change caught a fabricated
`0.000 Hz` reading when the mean was not computed, an aggregate error state
made invisible by the faster readings poll clearing the shared banner, history
that was never cleared, and invalid channels skewing the range.

---

## 10. Traps for maintainers

These are the non-obvious properties of the implementation. Each one is a way
a future change could break the feature while still compiling and passing a
casual read.

**MTR1 and MTR2 share the magic and the size, and their word meanings
collide.** Both records are 256 bytes and both carry `MTR1` in word 0 — only
**word 1** disambiguates them. The overlap is not benign: word 9 is
`capture_frames` in MTR1 but `first_basic_sequence` in MTR2, word 10 is header
errors versus last Basic sequence, word 11 is FIFO overflows versus the
composition word, and words 12/13 are packetizer/hub drops versus the
aggregate first-sample index. Calling an MTR1 accessor on an MTR2 record
**compiles and silently returns garbage** — there is no type-level guard. Any
tool that reads the record stream must dispatch on word 1 first. (A practical
demonstration: querying the SQLite stream with MTR1 channel offsets makes
aggregate rows appear as all-zero channels, which looks like a metrology fault
and is not.)

**The 15-block span check is modular, and the wrap case is deliberate.** The
decoder subtracts two `uint32_t` accessors, so `first = 0xFFFFFFF8`,
`last = 0x00000006` yields a span of 14 and is accepted. A reviewer who
"corrects" this to signed or 64-bit arithmetic breaks a valid case; a test
pins it.

**`meter_record_age_ms` is not basic-only.** The ingestor refreshes the
record timestamp for *every* accepted record, aggregates included. Against the
one-second staleness threshold, an arriving aggregate can therefore keep that
age looking fresh even if basic records have stalled. Only the aggregate's own
timestamp is format-exclusive. Anything that needs "age of the newest basic
record" must not use this field as-is — see future work.

**VHDL records cannot cross a mixed-language simulation boundary.** The
aggregator's input is a VHDL record, so the SystemVerilog unit testbench
drives it through a small test-only shim that flattens the record into scalar
ports. The shim is not part of the production hierarchy.

**`sequence` is a reserved word in VHDL-2008**, hence `result_sequence` in the
shared record. This cost a compile cycle during implementation.

**Every new PL source file must be added to the hardcoded Tcl file lists.**
The verification scripts do not glob; a new entity that is not registered will
silently not be simulated or synthesized.

---

## 11. Deployment coupling

- **PL and APU must ship together.** The PL emits MTR2; an older APU rejects
  it as an unknown format. The new APU keeps the v1 and v2 basic decoders, so
  previously stored streams remain readable.
- **PL and RPU must ship together** for the health registers, and the PL
  change requires a fresh bitstream-inclusive XSA.
- **APU and WEB**: the aggregate endpoint and IPC version are additive; the
  Web falls back cleanly when the endpoint reports nothing available.

Both record formats share one DMA channel and one SQLite record stream.
Consumers distinguish them by record word 1 — `0x00010002` basic,
`0x00020001` aggregate — and should expect roughly 15 basic records per
aggregate.

---

## 12. Non-goals and future work

Explicitly not implemented in this milestone: 10-minute and 2-hour
aggregation, UTC-aligned 10-minute boundaries, the standardized 10 s Class A
frequency measurement, harmonics, interharmonics, flicker, PQ events, power,
energy, demand, HLS, RPU aggregation, additional DMA channels, and any
RPMsg measurement payload.

Deferred but identified:

- **The 10 s frequency tier.** Until it exists, the aggregate frequency
  remains informative-only, and the API is built so it cannot be mistaken for
  a Class A result.
- **Aggregate gap counters and timestamp uncertainty over REST.** The ingestor
  counts aggregate sequence gaps and logs them, but the count is not carried
  in the IPC response, so only the basic gap count reaches health consumers.
  Timestamp uncertainty is likewise tracked but not exposed.
- **A basic-only record-freshness field.** See the `meter_record_age_ms`
  trap above: staleness detection would be sharper with a per-format age.
- **The 10-minute tier** will need UTC boundary alignment and aggregation
  quality propagation, which are genuinely new problems rather than more of
  the same: this tier's boundaries are defined purely by block counting, while
  the 10-minute tier must align to absolute time.

The provenance now proven end to end — block sequence, sample ranges, cycle
counts, configuration generation, lock flags, time quality, timestamp
uncertainty, and a tested eligibility gate — is what the next tier will
consume.
