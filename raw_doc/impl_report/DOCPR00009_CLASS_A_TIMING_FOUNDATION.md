# MSAP1 Class A basic measurement timing foundation

## Document purpose

This document describes the implemented "Class A Basic Measurement Timing
Foundation" milestone: the refactor that changed the MSAP1 metering pipeline
from a fixed 200 ms measurement interval to IEC 61000-4-30 style,
cycle-defined basic measurement blocks. It is the detailed implementation
report for the coordinated change across all four product repositories.

The document covers:

- the timing model: cycle-defined blocks, the two time domains, and the
  separation of nominal frequency, measured frequency, and cycle timing;
- the PL implementation: the 64-bit monotonic sample counter, the
  `grid_cycle_timing` component, the RMS engine changes, and the new `MTR1`
  format 2 record;
- the RPU configuration path: the nominal-frequency wire field and the new
  grid-timing registers;
- the APU implementation: the format-2 decoder, `BasicMeasurementBlock`
  timing metadata, the UTC measurement timebase, and time quality;
- the Web UI changes and the REST surface;
- the 128 kSPS default sample-rate change made in the same window;
- verification: simulation, builds, and on-hardware evidence;
- deployment coupling between the repositories;
- explicit non-goals and the intended next milestone.

Associated pull requests (all `feat/timing_foundation` → `main`):

```text
MSAP1_PL   #17  Class A basic measurement timing foundation (grid-cycle blocks)
MSAP1_RPU  #16  Nominal grid frequency configuration and 128 kSPS boot default
MSAP1_APU  #31  Class A timing foundation: cycle-defined basic blocks,
                measurement timebase, 128 kSPS default
MSAP1_WEB  #15  Nominal frequency selection, block-timing display, and
                simulator/tweak reorganization
```

The short normative summary of the timing model lives at
`applications/MSAP1_APU/docs/TIMING_MODEL.md`. The per-module register and
record contracts remain in the PL module READMEs
(`SourceData/DesignFile/MeterProcessing/README.md`,
`.../MeterCore/README.md`, `.../AdcConversion/README.md`). If a bit-level
detail differs between this report and those contracts or the source, the
contracts and the source are authoritative.

---

## 1. Introduction and motivation

IEC 61000-4-30 Class A defines the basic measurement time interval for
voltage magnitude, harmonics, and unbalance as **10 cycles at 50 Hz nominal /
12 cycles at 60 Hz nominal** — a *cycle-defined* interval that is
approximately, but deliberately not exactly, 200 ms. Aggregations (150/180
cycles, 10 min, 2 h) are built from these blocks.

Before this milestone, MSAP1 modeled the basic measurement as a fixed
200 ms interval:

- the PL RMS window was a configured **sample count** (6,400 samples =
  200 ms at 32 kSPS), derived on the APU from `rms.window_ms`;
- the decoded result was labeled `UpdatePeriod::ms200`;
- the record carried no sample-domain provenance, so future aggregation
  could not have proven block continuity.

This milestone replaced that semantic with cycle-defined blocks and put in
place all the timing infrastructure future aggregation needs, while
explicitly **not** implementing any aggregation, harmonics, flicker, or PQ
event detection. The electrical calculation set is unchanged: fundamental
RMS and mean per channel, plus the existing frequency measurement.

Two properties were treated as non-negotiable throughout:

1. **Gaplessness.** Block N+1 begins at exactly the sample after the last
   sample of block N. No ADC sample silently disappears between blocks, and
   the record format proves it (`first(N+1) = first(N) + count(N)`).
2. **Time-domain separation.** The measurement timebase (a free-running PL
   sample counter) is never stepped, paused, or reset by configuration or by
   Linux time synchronization. UTC is attached to measurements purely as a
   *mapping* maintained on the APU.

---

## 2. The timing model

### 2.1 Cycle-defined, not time-defined

```text
nominal 50 Hz  ->  basic block = 10 complete grid cycles
nominal 60 Hz  ->  basic block = 12 complete grid cycles
```

The nominal duration is approximately 200 ms, but 200 ms is never the
semantic definition. At 60.1 Hz a 12-cycle block is ~199.7 ms; at 59.9 Hz it
is ~200.3 ms. The sample count per block therefore varies from record to
record. This is intentional and observable in the shipped records (see
section 10.3).

The **declared nominal frequency is configuration, not measurement**. It is
set by the user (50 or 60 Hz only; everything else is rejected at every
layer), selects the cycle count, and is never inferred from the measured
frequency. The measured frequency continues to come from the unchanged PL
frequency estimator.

Three concepts, three owners:

| Concept | What it is | Where it lives |
| --- | --- | --- |
| Nominal frequency | 50/60 Hz configuration | Settings → RPMsg → PL `GRID_*` registers |
| Measured frequency | actual grid frequency | PL `meter_frequency` (unchanged by this milestone) |
| Cycle timing | block boundary tracking | PL `grid_cycle_timing` |

### 2.2 Two time domains

```text
Measurement time                     UTC time
----------------                     --------
PL 64-bit sample counter             CLOCK_REALTIME / CLOCK_TAI on the APU
increments per accepted ADC frame    disciplined by NTP or manual setting
never reset by configuration apply   may step or slew at any moment
never touched by time sync           attached to measurements via sync points
```

The bridge between the domains is a **sync point**: an atomic PL latch of
the sample counter, bracketed by APU clock reads:

```text
TimeSyncPoint {
    uint64 sample_counter;   // PL measurement domain
    int64  utc_ns;           // bracket midpoint, CLOCK_REALTIME
    uint64 uncertainty_ns;   // bracket width + elasticity-FIFO bound
}
```

A UTC timestamp for any sample is derived linearly from the newest sync
point. When Linux corrects its clock, only the mapping changes; sample
indices, block boundaries, and record contents are unaffected.

### 2.3 Time quality

```text
startup                          -> Unsynchronized
valid sync point recorded        -> Synchronized
newest sync older than threshold -> Holdover      (default 30 s = 3 missed
new sync point                   -> Synchronized   10 s refresh periods)
```

Time quality is **metadata**. An unavailable or degraded UTC mapping never
marks the electrical measurement invalid; `MeasurementQuality` (per channel)
and `TimeQuality` (per block) are independent fields.

---

## 3. Ownership across the repositories

```text
ADC / PL raw simulator
      |
      v
MSAP1_PL   capture -> conversion (64-bit sample counter in TUSER)
           -> RMS engine  <- frame_closes_block marker
           -> frequency engine -> shared zero-cross detector
                                        |  combinational crossing view
           -> grid_cycle_timing  <------+
           -> MeterResultHub (MTR1 format 2) -> DMA to Linux
           -> waveform branch (latches the sample counter for correlation)
      |
MSAP1_RPU  R5 core 0: validates config, derives cycles-per-block,
           programs GRID registers, readback-verifies. No MTR1 knowledge,
           no DMA, no zero-cross logic. Core 1 unchanged (comm only).
      |
MSAP1_APU  DMA ingest -> format-2 decoder -> BasicMeasurementBlock timing
           SQLite record stream (unchanged publication boundary)
           MeasurementTimebase (UTC sync authority, time quality)
           settings authority (nominal frequency, sample rate)
           REST: /api/v1/meter/readings exposes block timing + time quality
      |
MSAP1_WEB  nominal frequency selector, cycle-aware dashboard,
           time-quality pill, developer-side simulator/tweak controls
```

Responsibilities in one line each:

- **PL** owns cycle timing and the monotonic measurement counter.
- **RPU** owns configuration/control transport and validation; it is a pure
  conduit for the new field.
- **APU** owns the UTC sync authority, decoding, storage, and the REST
  surface.
- **WEB** owns presentation and user configuration entry.

---

## 4. PL implementation (`MSAP1_PL`)

All changes stay inside the single `MeterCore` module-reference boundary; no
block-design, address-map, or device-tree changes were required. New RTL
follows the established observational rule: grid timing has no `ready`
output and can never backpressure capture, RMS, or record production.

### 4.1 64-bit monotonic sample counter

`adc_conversion.vhd` widened its per-frame sequence from 32 to 64 bits. The
counter increments once per accepted 8-channel frame, from PL reset only.
Configuration apply, capture stop/start, and Linux time changes never touch
it. At 128 kSPS a 64-bit counter wraps after ~4.5 million years; the old
32-bit counter wrapped in ~9 hours at that rate.

Placement preserved every existing consumer: the low word stays in
`TUSER[31:0]`; the high word rides in the previously unused `TUSER[105:74]`
window, so the TUSER width (384) and the elasticity-FIFO generics are
unchanged. The layout is now documented centrally in `metering_pkg.vhd`:

```text
TUSER[31:0]    sample index, low word (was the 32-bit sequence)
TUSER[63:32]   active configuration generation
TUSER[71:64]   valid channel mask
TUSER[72]      saturation sticky
TUSER[73]      source-packet TLAST
TUSER[105:74]  sample index, high word          <- new
TUSER[127:106] unused
TUSER[383:128] eight raw 32-bit ADC words (waveform branch)
```

Because the counter value travels *with* each frame, every tap along the
pipeline (waveform branch before the elasticity FIFO, RMS/frequency/grid
timing after it) observes the same index for the same frame, regardless of
FIFO occupancy. The conversion AXI register `SAMPLE_SEQUENCE` (0x40) keeps
exposing the low 32 bits.

### 4.2 One zero-cross detector, two views

The milestone required cycle timing without duplicating the validated
zero-crossing implementation. `meter_zero_crossing.vhd` was refactored so a
single shared expression drives two views of the same decision:

- the **registered view** (unchanged behavior): `crossing_valid_o` plus the
  bracketing samples, one clock after the accepted frame — consumed by the
  frequency estimator exactly as before;
- a new **combinational "now" view**: `rising_crossing_now_o` /
  `falling_crossing_now_o`, valid *during* the accepted frame — consumed by
  grid-cycle timing.

Falling-crossing detection (new, for half-cycle boundaries) mirrors the
rising path with its own hysteresis arming: rising crossings arm below
`-hysteresis`, falling crossings arm above `+hysteresis`. Because both views
share one expression, they cannot disagree about whether a frame crossed
zero. `meter_frequency.vhd` re-exports the combinational view and the
per-frame reference-validity signal, so grid timing uses the same qualified
crossings, the same reference channel (CH6/VLA), and the same hysteresis as
the frequency measurement.

### 4.3 `grid_cycle_timing`

New entity `MeterProcessing/grid_cycle_timing.vhd`, contract constants in
`MeterCommon/grid_timing_pkg.vhd`. Inputs: accepted-frame strobe (the same
`engine_valid` the RMS engine uses), the 64-bit sample index from TUSER, the
combinational crossing view, the shadow grid configuration, the fallback
window, and the shared APPLY toggle.

**Block-close rules**, evaluated combinationally on every accepted frame
(the three causes are mutually exclusive):

```text
close_locked    locked   and rising crossing and cycles+1 >= cycles_per_block
close_relock    unlocked and rising crossing
close_fallback  unlocked and no crossing and samples+1 >= fallback_window
```

**Boundary convention**: the frame carrying the closing crossing is the
*last* frame of its block. The next block therefore starts at
`closing_index + 1`, which makes consecutive blocks gapless by construction
rather than by bookkeeping.

**Lock model**:

- Startup and every APPLY begin unlocked. The first qualified rising
  crossing closes the initial partial block (`close_relock`) and locks —
  so alignment to the grid happens within one cycle of a usable reference.
- The lock drops immediately when the reference becomes unusable, or when
  no qualified crossing has arrived for a quarter of the fallback window
  (a divider-free `window >> 2`, i.e. ~2.5 nominal cycles at 50 Hz, ~3 at
  60 Hz).
- While unlocked, blocks close on the fallback sample count so records keep
  flowing on a dead grid; the next qualified crossing closes the running
  partial block early and re-locks. Every block that did not close on a
  counted crossing is flagged `free_run_fallback`; blocks may be short
  around a re-lock, which the flags and the sample count make visible
  rather than hiding.

**Provenance outputs** describe the most recently closed block and stay
stable until the next close (comfortably longer than the RMS calculation
latency, so the result hub always samples coherent values): first sample
index, cycle count, nominal frequency, and the flags
`cycle_locked` / `free_run_fallback` / `first_block_after_apply`.

**Future PQ hooks** (generated and unit-tested, consumed by nothing yet, as
the milestone required): `cycle_boundary`, `half_cycle_boundary` (from the
falling-crossing view), and a free-running `cycle_sequence`.

Configuration commits on the same APPLY toggle as the RMS and frequency
engines, in the same single-cycle copy discipline, so one block can never
span two configuration generations.

### 4.4 RMS engine changes

Two changes in `meter_rms.vhd`, both minimal by design:

1. **Pipeline-carried close marker.** The naive approach — grid timing
   strobing the RMS engine — would have required exact latency matching
   between two independently pipelined modules and would break silently if
   either pipeline changed. Instead, `frame_closes_block_i` is valid on the
   same clock as the accepted frame and is **registered through the RMS
   input pipeline together with the frame** (`sample_stage_closes` →
   `square_stage_closes`). The accumulate stage then closes the window when
   the marker arrives with its frame. Grid timing and the RMS engine
   therefore agree on block membership by construction, at any pipeline
   depth.

   In cycle mode (`cycle_mode_i = '1'`, driven by the grid active enable)
   the marker is the only close source; the legacy sample-count comparison
   remains the close source when cycle timing is disabled, and is the
   fallback semantics *inside* grid timing when unlocked.

2. **Actual-count divisor.** The engine previously divided by the
   *configured* window (`snapshot_window <= active_window_samples`), which
   is wrong for variable-length blocks. It now snapshots the *actual*
   accumulated count (`sample_count + 1`). All downstream arithmetic — mean
   divide, variance product, `N²` denominator — and the record's word 6
   follow automatically from that one assignment. In fixed-window mode the
   actual count equals the configured window on every close, so legacy
   behavior is bit-identical (regression-checked by the unchanged expected
   values in `voltage_rms_tb` and `meter_core_tb`).

### 4.5 New processing registers

Three registers in the RPU-owned processing block (base `0xB0050000`), in
the previously free region; shadow commits on the existing `CONTROL.APPLY`
toggle at offset 0x08:

| Offset | Register | Layout |
| --- | --- | --- |
| `0x6C` | `GRID_SHADOW_CONFIG` | [7:0] cycles per block, [15:8] nominal Hz, [16] cycle-timing enable |
| `0x70` | `GRID_ACTIVE_CONFIG` | committed readback, same layout |
| `0x74` | `GRID_STATUS` | [0] locked, [1] reference usable, [2] enabled, [15:8] cycles in the open block |

Reset default is `0x00013C0C` (enabled, 60 Hz, 12 cycles); it only governs
behavior between reset and the first apply, because the RMS engine is
disabled until software configures it. The PL does not validate the
nominal/cycle pairing — the RPU derives and owns that rule.

`SHADOW_WINDOW_SAMPLES` (0x18) keeps its offset and encoding but its meaning
narrowed: it is now the **free-run fallback window** (and the whole close
source only when cycle timing is disabled). Software programs it to the
nominal block length.

### 4.6 `MTR1` record format 2

The record stays 256 bytes / 64 little-endian words with the same magic.
Word 1 (format) bumped `0x00010001 → 0x00010002`. The APU decoder registry
keys on word 1, so this is an additive, versioned change; format-1 records
in stored streams remain decodable.

Full word map (changes versus format 1 marked `*`):

| Word | Content |
| ---: | --- |
| 0 | magic `MTR1` (`0x3152544D`) |
| 1 | `*` format `0x00010002` |
| 2 | byte length = 256 |
| 3 | result sequence (= basic block sequence) |
| 4 | configuration generation |
| 5 | sample rate (Hz) |
| 6 | `*` **actual** sample count of this block (was configured window) |
| 7 | valid channel mask |
| 8 | result status (bit 0 arithmetic overflow) |
| 9–14 | capture frame count, header errors, FIFO overflows, packetizer drops, hub drops, ADC alerts |
| 15 | `*` timing word: [7:0] nominal Hz, [15:8] cycle count, [16] `cycle_locked`, [17] `free_run_fallback`, [18] `first_block_after_apply` |
| 16–55 | eight channels × 5 words: mean lo/hi, raw RMS count, RMS lo/hi (micro-units) |
| 56–59 | frequency: millihertz, status word, averaged Q16 period, measurement sequence |
| 60 | `*` first sample index of this block, bits [31:0] |
| 61 | `*` first sample index, bits [63:32] |
| 62–63 | reserved (zero) |

`last_sample_index` is deliberately not recorded:
`last = first + count − 1`. After format 2, exactly two spare words remain;
future aggregate records are expected to define their own format words
through the same registry mechanism rather than squeeze into this one.

### 4.7 Waveform correlation alignment

`meter_waveform.vhd` previously counted accepted frames with a private
64-bit counter exposed through the Linux-owned correlation registers. It now
latches the **TUSER-carried conversion sample index** instead. Consequences:

- the correlation registers, the kernel driver, and the ioctl ABI are
  unchanged — only the semantics of the "frame sequence" field sharpened to
  "conversion-domain sample index";
- a correlation read now maps **the same counter that MTR1 words 60/61
  reference** to `CLOCK_TAI`/`CLOCK_REALTIME`, which is exactly what the
  APU UTC mapping needs;
- the residual uncertainty from the waveform tap sitting before the
  16-frame elasticity FIFO is bounded (≤ 0.125 ms at 128 kSPS, ≤ 0.5 ms at
  32 kSPS) and is added into the sync-point uncertainty rather than
  ignored.

### 4.8 PL verification

Simulation (all `PASS`, run via the maintained batch Tcl):

- `grid_cycle_timing_tb` (new, unit level): exactly 10 cycles per block at
  50 Hz and 12 at 60 Hz; off-nominal cycles (7- and 9-frame periods)
  regroup by complete cycles with proportional sample counts; every close
  satisfies `first(N+1) = first(N) + count(N)` including across
  fallback/re-lock transitions; reference loss closes on the fallback
  window with correct flags; APPLY restarts cleanly with
  `first_block_after_apply`.
- `meter_core_tb` (extended): the legacy fixed-window scenario now asserts
  the format-2 header, timing word, and first-sample chain (1 → 5 → 9);
  a new end-to-end cycle-mode scenario drives a synthetic grid waveform on
  CH6 over the real four-lane AD7771 serial interface and asserts word-exact
  records for a re-lock block (21 samples) followed by locked 2-cycle
  blocks (40 samples each) with gapless first-sample indices (13 → 34 →
  74).
- `voltage_rms_tb`, `meter_packet_tb`, `adc_conversion_tb`,
  `meter_frequency_tb` (regression; the frequency path is untouched),
  `adc_simulator_tb`.
- `check_metering_module_references.tcl` (interface inference) and
  `check_metering_synthesis.tcl MeterCore_Wrapper` (focused synthesis with
  timing report) both pass.

---

## 5. RPU implementation (`MSAP1_RPU`)

The RPU remains a pure configuration conduit; it gained no measurement
logic, no MTR1 knowledge, and no DMA involvement.

### 5.1 Wire ABI

`msap1_meter_config_payload` gained one trailing field (packed size
172 → 176 bytes; header + payload = 192 of the 256-byte frame budget; wire
version stays 2; the header copy is byte-identical between the RPU and APU
repositories, verified by `diff`):

```c
/* Declared nominal grid frequency in hertz. Only 50 or 60 is valid. ... */
uint32_t nominal_frequency_hz;
```

The cycle count is deliberately **not** on the wire — it is derivable, and
the no-redundant-fields rule applies. The compile-time size gate in
`r5c0_service.cpp` moved from 172 to 176.

### 5.2 Validation, derivation, and the apply transaction

- `valid_configuration()` rejects any nominal frequency other than 50/60.
- `cycles_per_block()` derives 10 or 12 (constexpr, with the IEC reference
  in a comment).
- `configure()` packs and writes `GRID_SHADOW_CONFIG` alongside the other
  processing shadows, commits everything with the single existing APPLY
  write, and **readback-verifies `GRID_ACTIVE_CONFIG`** in the same
  `ReadbackMismatch` pattern as the five frequency registers — an
  unverified register would silently accept an unlatched value.
- `status()` reads `GRID_STATUS` into the internal `Status` struct. The
  RPMsg health payload was intentionally not extended in this milestone
  (only 42 bytes of frame headroom remain there).
- Cycle timing is always enabled when configuring; the PL's own fallback
  covers reference loss, so no separate enable policy is needed on the RPU.

### 5.3 Boot default (see also section 9)

`main.cpp` now initializes the AD7771 at `Sps128000` with the existing
Sinc5 / high-resolution / 8.192 MHz MCLK profile; internal configuration
defaults follow (25,600-sample fallback window; 1 s frequency averaging
window at 128 kSPS). Both R5 ELFs build cleanly via
`vitis -s scripts/build_r5_apps.py -- all`; no platform/XSA regeneration was
needed for compilation because the new registers are offsets inside the
existing processing base.

---

## 6. APU implementation (`MSAP1_APU`)

### 6.1 Types

New header `common/msap1/meter/meter_timing.hpp`:

```cpp
enum class NominalFrequency : std::uint8_t { Hz50 = 50, Hz60 = 60 };
enum class MeasurementPeriod : std::uint8_t { Basic, Cycles150_180, Min10, Hour2 };
enum class TimeQuality : std::uint8_t { Unsynchronized, Synchronized, Holdover };
constexpr std::uint16_t cycles_per_basic_block(NominalFrequency);
struct BlockTiming { /* sequence, generation, first_sample_index,
    sample_count, cycle_count, nominal_frequency, cycle_locked,
    free_run_fallback, first_block_after_apply, time_quality,
    optional utc_start */ };
```

`UpdatePeriod` (with its `ms200` value) was **removed outright** rather than
extended: a repository-wide search proved `MeterData` had no consumers
beyond the ingestor and tests, so the six-slot store shrank to the four
`MeasurementPeriod` slots cheaply, and the `duration()`/`update_period()`
free functions disappeared — a Basic block has no fixed duration; its actual
duration is `sample_count / sample_rate`, already carried per-update by
`SampleWindow`. Names like `200ms` no longer appear as standards-oriented
period identifiers anywhere in the tree.

### 6.2 Record accessors and decoders

`meter_record.hpp` gained `meter_periodic_format_v2 = 0x00010002`, a
`record_format()` accessor, dual-format header validation, and typed
accessors for the new words (`block_sample_count()`, `first_sample_index()`,
and the decoded word-15 timing struct). The decoder registry (keyed on
word 1) now registers two decoders:

- `0x00010001` — kept for stored streams; produces
  `MeasurementPeriod::Basic` without sample-domain timing;
- `0x00010002` — produces `Basic` plus a fully populated `BlockTiming`.

`MeterData` remains a storage/publication abstraction; no aggregation logic
was added.

### 6.3 Measurement timebase and sync source

`common/msap1/meter/measurement_timebase.{hpp,cpp}` implements the
`TimeSyncPoint` mapping and the quality state machine described in
section 2. It is thread-safe (one writer — the sync refresher; readers on
the decode path), uses `std::chrono::steady_clock` for staleness so UTC
steps cannot corrupt the state machine, and derives timestamps with exact
integer arithmetic.

The sync source reuses the waveform correlation machinery
(`WaveformCapture::time_sync()`): a `CLOCK_REALTIME` bracket around the
existing atomic latch ioctl, midpoint as `utc_ns`, bracket width plus the
elasticity-FIFO bound as `uncertainty_ns`. `CaptureCoordinator` refreshes
the sync point every 10 s in its run loop and stamps decoded blocks with
`time_quality` and `utc_start = utc_for_sample(first_sample_index)` after
decode. No kernel or ioctl change was required (see section 4.7).

### 6.4 Ingest validation

`record_ingestor.cpp` accepts both formats. For format 2 it **replaces** the
old `window_samples == configured` equality check — meaningless for
variable-length blocks — with sample-range continuity:

```text
first_sample_index(N+1) == first_sample_index(N) + sample_count(N)
```

plus the existing sequence-continuity and generation checks. A mismatch is
counted and logged like a sequence gap and re-baselines, mirroring the
established gap-handling behavior. The ingestor also took the pre-existing
`MeterRecordSource` interface in its constructor, which made the new
continuity behavior unit-testable without device access.

### 6.5 Settings and wire encoding

- `MeteringSettings.nominal_frequency_hz` (default 60) with inline
  validation accepting only 50/60, following the established
  `RmsSettings::validate()` pattern; present in
  `config/settings/factory-defaults.json`; carried through
  `to_meter_configuration()` with default-when-absent for older
  conversion-file documents (schema version unchanged).
- `prepare_meter_configuration()` sets the wire field and derives
  `rms_window_samples = sample_rate × cycles_per_block / nominal` — exact
  at every supported rate, and 25,600 for **both** nominals at 128 kSPS
  (6,400 at 32 kSPS), a convenient coincidence of the supported rates.
  `rms.window_ms` remains in the schema for compatibility but no longer
  feeds this derivation (documented as superseded).
- The FNV-1a configuration fingerprint covers the whole payload, so adding
  the field automatically changes the generation, which drives PL re-apply
  and the record-generation checks with no additional code.

### 6.6 IPC and REST surface

- `InfoResponse` gained `time_quality`; the acquisition IPC version bumped
  16 → 17.
- `GET /api/v1/meter/readings` gained an optional `timing` object (omitted
  while the latest record is format 1):

```json
"timing": {
  "block_sequence": 17254,
  "first_sample_index": 331990790,
  "sample_count": 25605,
  "cycle_count": 12,
  "nominal_frequency_hz": 60,
  "cycle_locked": true,
  "free_run_fallback": false,
  "time_quality": "synchronized"
}
```

- Settings routes needed no code (glaze reflection over the struct).

### 6.7 APU verification

- Host build (x86_64, Release): warning-clean under
  `-Wall -Wextra -Wpedantic`; `ctest` **15/15 pass**, including the new
  `measurement_timebase_test` (state machine, linear mapping,
  UTC-step-changes-mapping-only) and extended protocol / meter-data /
  acquisition-architecture / settings tests (wire layout 176 bytes with
  offset asserts, 50/60 accepted and 55 rejected, v2 decode with timing
  metadata, v1 still decodes, gapless and gap-detected ingest, u32
  sequence wrap).
- aarch64 cross build: clean (compile verification; binaries are not
  host-runnable).
- `tests/method/test_adc_rpmsg_procedure.md` gained section 8, the
  deterministic on-target procedure: 50/60 nominal checks, off-nominal
  simulator frequencies, continuity, the 55 Hz rejection, time-quality
  transitions, and the dead-reference fallback/re-lock check.

---

## 7. Web implementation (`MSAP1_WEB`)

- **Configuration → Meter**: "Nominal grid frequency" select (50/60 Hz)
  with a live "Basic measurement block: 10/12 cycles" hint, saved through
  the existing whole-document `PUT /api/v1/settings/active` flow together
  with the zero-crossing form. The ADC simulator's "Signal frequency (Hz)"
  input also moved here (it only writes `adc.simulator.frequency_hz`, so
  the Meter form can never clobber the simulator channel table).
- **Dashboard**: the hardcoded "Mean-corrected 200 ms RMS" / "Update
  period: 200 ms" strings became timing-aware — with a format-2 record the
  captions read from `timing` (e.g. "12-cycle basic block", "Basic block:
  12 cycles (60 Hz nominal)") and a time-quality pill shows
  `Time synchronized/holdover/unsynchronized`, right-aligned with the
  basic-block caption (a follow-up fix grouped the pill and caption into
  one flex container; the heading row is `space-between`, so a third
  direct child had floated to the center).
- **Developer**: new "Tweak" sub-tab with the sample-rate selector
  (1–128 kSPS, saved via the settings document, restart-capture warning),
  and the relocated ADC Simulator pane (source selection, health, channel
  table) — source switching is a developer diagnostic, not a user setting.
  Developer sub-tabs are now Overview / Tweak / ADC Simulator / About /
  Logs; Configuration keeps Meter and Waveform.
- `api.ts` typed `metering.nominal_frequency_hz`, `MeterBlockTiming`, and
  the optional `timing` on `MeterReadings`.
- Build: `npm ci && npm run build` clean (tsc + vite; requires Node ≥
  20.19).

---

## 8. What the simulator path provides for testing

The PL raw ADC simulator (synthesized into the bitstream, not a testbench
model) already supported arbitrary frequencies via the Q0.32 phase step the
APU computes from `simulator_frequency_millihz`. No simulator changes were
needed: setting 49.9/50/50.1 or 59.9/60/60.1 Hz through the settings
document deterministically exercises cycle/block boundaries without
hardware, which is exactly what target procedure section 8 uses. Note the
simulator's CH6 is Va — the frequency/cycle reference — so zeroing its peak
is the deterministic way to exercise the fallback/re-lock path.

---

## 9. Default sample rate change to 128 kSPS

Made in the same merge window at the product owner's request, after the
128 kSPS operating point was validated end-to-end on hardware:

- **APU**: factory settings ship `sample_rate_hz: 128000`; the
  `load/prepare_meter_configuration` default arguments and the settings
  test follow; README/AGENTS wording updated (25,600-frame fallback window
  at the default rate; waveform retention ≈ 32 s at 128 kSPS vs 131 s at
  32 kSPS).
- **RPU**: boot configuration at `Sps128000` (decimation derives from
  `MCLK / (divisor × rate)`, so the existing profile machinery needed no
  change); internal defaults aligned; AGENTS/docs no longer state a
  32 kSPS boot default.
- **WEB**: the Tweak selector defaults/labels accordingly.

All supported rates (1/2/4/8/16/32/64/128 kSPS) remain selectable, and the
basic-block semantics are rate-independent (section 10.3 note): the block
is always 10/12 cycles; only the samples-per-block scale with the rate.

An operational note that motivated the Tweak UI: a rate selected through
the temporary diagnostic path is **not persisted** — the persisted profile
(now 128 kSPS from the factory) is restored on daemon restart. Selecting
the rate through the settings document is the durable path.

---

## 10. Verification summary and hardware evidence

### 10.1 Per-repository verification

| Repo | Verification | Result |
| --- | --- | --- |
| MSAP1_PL | 8 testbenches (incl. 2 new/extended), module-reference inference, focused synthesis | all PASS |
| MSAP1_RPU | `vitis` build of R5c0/R5c1 | success |
| MSAP1_APU | host `ctest` (15 tests), aarch64 cross build, warning-clean | 15/15, clean |
| MSAP1_WEB | `tsc -b && vite build` | clean |

### 10.2 On-hardware validation (production board, live grid)

Device: `msap1` (KR260 carrier, AD7771 front end), physical three-phase
input ~117 V per phase, running the new PL bitstream, R5 firmware, APU
package, and web frontend at 128 kSPS.

Health at inspection time: services active, 0 failed units; configured
128000 vs measured 128001 frames/s; **0 DMA errors, 0 invalid records, 0
sequence gaps, 0 FIFO overflows, 0 header errors** across 17k+ records
(~57 minutes); PL generation match; frequency 59.986–59.991 Hz.

### 10.3 Raw record evidence

Words decoded directly from the on-device SQLite record stream (byte-level,
independent of the decoding software):

```text
seq=17247  fmt=0x00010002  count=25606  nominal=60  cycles=12  locked  first=331811545
seq=17248  fmt=0x00010002  count=25606  nominal=60  cycles=12  locked  first=331837151  GAPLESS
seq=17249  fmt=0x00010002  count=25607  nominal=60  cycles=12  locked  first=331862757  GAPLESS
seq=17250  fmt=0x00010002  count=25606  nominal=60  cycles=12  locked  first=331888364  GAPLESS
seq=17251  fmt=0x00010002  count=25607  nominal=60  cycles=12  locked  first=331913970  GAPLESS
seq=17252  fmt=0x00010002  count=25606  nominal=60  cycles=12  locked  first=331939577  GAPLESS
seq=17253  fmt=0x00010002  count=25607  nominal=60  cycles=12  locked  first=331965183  GAPLESS
seq=17254  fmt=0x00010002  count=25605  nominal=60  cycles=12  locked  first=331990790  GAPLESS
```

Reading this table against the milestone's acceptance intent:

- every block is exactly 12 cycles at the declared 60 Hz nominal;
- the sample count *varies* (25605–25608 across the session) and matches
  `128001 × 12 / f_grid` for the concurrently measured 59.98–59.99 Hz —
  the block is tracking the grid, not a stopwatch;
- `first(N+1) = first(N) + count(N)` holds on every consecutive pair;
- the 64-bit counter value (~331.9 M) matches the capture uptime at
  128 kSPS and was never disturbed by configuration or time changes;
- `cycle_locked` is set and `free_run_fallback` clear throughout, on real
  (not simulated) grid input.

This is the sequence shape the milestone specified as its expected result,
with the sole difference that the validated hardware runs at 128 kSPS
rather than the 32 kSPS used in the planning examples.

### 10.4 Sample-rate independence

Because block boundaries derive from zero crossings of the actual waveform,
the cycle grouping is independent of the input sample rate; only these
implementation details scale, and all of them scale automatically through
the existing configuration transaction: the fallback window (derived
per-rate on the APU), the crossing-staleness threshold (window/4 in
samples), and the UTC mapping slope. A rate change bumps the configuration
generation, restarts block tracking (`first_block_after_apply`), and can
never produce a block spanning two rates. The one bounded caveat: a sync
point captured before a rate change extrapolates with the new rate until
the next 10 s refresh; measurement data is unaffected.

---

## 11. Deployment coupling

The four changes ship as one product image and must be deployed together:

- **PL ↔ RPU**: `configure()` readback-verifies `GRID_ACTIVE_CONFIG`;
  against an old bitstream the register does not exist and configuration
  fails with `ReadbackMismatch` (deliberately loud rather than silently
  unlatched).
- **PL ↔ APU**: the new PL emits only format-2 records; an old APU would
  reject them as unknown-format. The new APU keeps the format-1 decoder,
  so previously stored streams remain readable.
- **APU ↔ WEB**: the readings `timing` object and IPC version 17 are
  additive; the web falls back to the legacy wording when `timing` is
  absent.
- Existing persisted settings documents remain valid: absent
  `nominal_frequency_hz` defaults to 60 Hz everywhere, and the factory
  128 kSPS applies to fresh/reset documents only.

Build chain for a release: PL implementation run → new bitstream-inclusive
XSA → `make_PL.sh && make_mconf.sh && make_RPU.sh && make_yocto.sh`, then
target procedure section 8.

---

## 12. Non-goals honored and next milestone

Explicitly not implemented, per the milestone definition: 150/180-cycle,
10-minute, and 2-hour aggregation; UTC boundary alignment and aggregation
quality propagation; harmonics/interharmonics; flicker;
dip/swell/interruption/RVC detection; power/energy/demand changes; PTP,
GPS/PPS, or PLL-grade clock discipline. The frequency measurement
implementation was preserved untouched, and `half_cycle_boundary` feeds
nothing yet by design.

The next milestone consumes these blocks:

```text
15 × BasicMeasurementBlock  ->  150/180-cycle (~3 s) Class A aggregate
```

with the provenance needed for it — block sequence, sample ranges, cycle
counts, configuration generation, lock flags, and time quality — already
proven trustworthy end to end.
