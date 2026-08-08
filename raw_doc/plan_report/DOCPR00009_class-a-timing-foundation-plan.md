# Implementation Plan — Class A Basic Measurement Timing Foundation

Milestone: refactor the metering pipeline so the basic measurement unit is a
cycle-defined block (10 cycles @ 50 Hz, 12 cycles @ 60 Hz), not a 200 ms
interval. Infrastructure only — no aggregation, harmonics, flicker, or PQ
events.

Repos touched: `MSAP1_PL`, `MSAP1_RPU`, `MSAP1_APU`, `MSAP1_WEB`.
Branches `feat/timing_foundation` already exist (clean, == main) in
MSAP1_PL and MSAP1_RPU; create matching branches in MSAP1_APU and MSAP1_WEB.

---

## 0. Current state (verified in code)

| Fact | Location |
|---|---|
| "200 ms" exists only in software; PL window is a **sample count** register (default 6400 = 200 ms @ 32 kSPS) | `MSAP1_PL .../MeterProcessing/meter_rms.vhd:74,201`, regs `0xB0050000+0x18` |
| APU derives the window as `sample_rate_hz * rms_window_ms / 1000` | `MSAP1_APU/common/msap1/meter/meter_config.cpp:165-170` |
| `UpdatePeriod::ms200` hardcoded as the MTR1 decode result | `MSAP1_APU/common/msap1/meter/meter_data.cpp:316`, enum at `meter_data.hpp:17-24` |
| PL already has a validated zero-cross detector (CH6/Va, hysteresis-armed, rising edge) and frequency estimator | `meter_zero_crossing.vhd`, `meter_frequency_estimator.vhd` |
| Per-frame sample sequence is only **32-bit** (wraps ~37 h @ 32 kSPS); estimator works around it with mod-2^48 math | `adc_conversion.vhd:65,184`, `meter_frequency_pkg.vhd:105-114` |
| A 64-bit free-running PL tick + 64-bit frame sequence already exist, but only in the Linux-owned waveform correlation block (`0xB0070000`), latched atomically and bracketed by CLOCK_TAI reads | `meter_waveform.vhd:68-69,247-254`, `waveform_capture.cpp:35-51,661-665` |
| MTR1 record: 256 bytes / 64 words, magic `MTR1`, format word `0x00010001`; **free words: 15 and 60–63 only** | `MeterResultHub_Wrapper.vhd:79-144`, `MSAP1_APU/common/msap1/meter/meter_record.hpp` |
| APU decoder registry keys on record word 1; adding a new format = registering a new decoder | `meter_data.cpp:328-354`, extension demo `tests/meter_data_test.cpp:148-165` |
| RPMsg config payload `msap1_meter_config_payload` is 172 B packed; frame cap 256 B ⇒ ~68 B headroom; wire version stays 2; APU/RPU header copies must stay byte-identical | `rpu_control_protocol.h:176-200`, static_asserts `r5c0_service.cpp:17-28` |
| RPU is a pure config conduit: validates, writes PL shadow regs, applies via toggle, verifies readback. Processing reg offsets `0x6C–0xFC` are free | `MSAP1_RPU/common/src/metering.cpp:9-43,130-216` |
| Ingestor rejects records when `window_samples != wire.rms_window_samples` — incompatible with variable-length cycle blocks | `record_ingestor.cpp:59-69` |
| `MeterData` has **no consumers** today (only the ingestor writes it); publication is the raw record via SQLite stream + `latest_record_` → REST | `record_ingestor.cpp:88-95`, `meter_routes.cpp:85-126` |
| Settings: glaze-reflected `ProductSettings`; whole-doc `PUT /api/v1/settings/active`; no `nominal frequency` concept anywhere in the tree | `metering_settings.hpp`, `MSAP1_WEB/src/api.ts:110-204` |
| ADC simulator is PL RTL; frequency set via Q0.32 `phase_step` computed by APU from `simulator_frequency_millihz` — off-nominal frequencies already supported | `adc_simulator.vhd:334-359`, `meter_config.cpp:302-360` |

Convenient coincidence: at 32 kSPS both nominal blocks are 6400 samples
(50 Hz → 10 × 640; 60 Hz → 12 × 533.33).

---

## 1. Cross-repo contracts (design first, sign off before coding)

### 1.1 Settings (APU JSON ⇄ WEB)

Add to `MeteringSettings` (`common/msap1/settings/definition/metering_settings.hpp`),
not inside `FrequencyConfig` — nominal frequency is metrology-wide
configuration, distinct from the zero-cross *measurement* config:

```cpp
enum class NominalFrequency : std::uint8_t { Hz50 = 50, Hz60 = 60 };
// serialized as integer "nominal_frequency_hz": 50 | 60
```

- `validate()` throws unless value ∈ {50, 60}.
- Factory default: **60 Hz** (matches current simulator/test defaults — confirm per market).
- `rms.window_ms` stays in the schema for compatibility but is **no longer
  the source of the basic block**; the fallback window is derived from
  nominal frequency (see 1.3). Document as superseded.

### 1.2 RPMsg APU → RPU (wire version stays 2)

Append one field to `msap1_meter_config_payload` in **both** byte-identical
header copies:

```c
	uint32_t nominal_frequency_hz;   /* 50 or 60; drives cycles-per-block */
```

- Size 172 → 176 (frame 16+176 = 192 ≤ 256). Update the static_assert at
  `r5c0_service.cpp:19` and extend `MSAP1_APU/tests/protocol_test.cpp`.
- RPU validates ∈ {50, 60} in `valid_configuration()` and derives
  `cycles_per_block` (50→10, 60→12). Cycle count is *not* on the wire
  (derivable — per the no-redundant-fields rule).
- `generation` fingerprint (FNV-1a over payload, `meter_config.cpp:364`)
  automatically changes when the field changes — re-apply and MTR1
  generation checks come for free.

### 1.3 PL registers (processing block `0xB0050000`, free range 0x6C+)

New shadow/active pair + status, committed by the **existing** apply toggle
(`CONTROL[0]` @ `0x08`) so a block never spans two config generations:

| Offset | Register | Layout |
|---|---|---|
| `0x6C` | `GRID_SHADOW_CONFIG` | [7:0] cycles_per_block, [15:8] nominal Hz (50/60), [16] cycle_timing_enable |
| `0x70` | `GRID_ACTIVE_CONFIG` | readback of the above |
| `0x74` | `GRID_STATUS` | [0] reference locked, [15:8] cycles elapsed in current block |

`SHADOW_WINDOW_SAMPLES` (`0x18`) keeps its meaning but becomes the
**fallback / free-run window**: APU now computes it as
`sample_rate * cycles_per_block / nominal_hz` (exactly 6400 at 32 kSPS for
both nominals) instead of from `window_ms`.

No new AXI base address, no BD change, no address-map/device-tree churn —
everything stays inside the existing MeterCore module-reference boundary.

### 1.4 MTR1 record — new format word `0x00010002`

Record stays 256 B / 64 words. Bump word 1 `0x00010001 → 0x00010002` and
register a **new** APU decoder; keep the v1 decoder registered (stored
streams may contain v1 records). Changes vs v1:

| Word | v1 meaning | v2 meaning |
|---|---|---|
| 1 | `0x00010001` | `0x00010002` |
| 6 | configured window samples | **actual sample count of this block** |
| 15 | zero | timing word: [7:0] nominal Hz, [15:8] cycle_count, [16] cycle_locked, [17] free_run_fallback, [18] first_block_after_apply, rest 0 |
| 60 | reserved | first_sample_index low 32 |
| 61 | reserved | first_sample_index high 32 |
| 62–63 | reserved | reserved (0) — last spare words |

`last_sample_index` is intentionally omitted (`first + count − 1`).
Block sequence = existing word 3; config generation = existing word 4.
Time quality is **not** in MTR1 — PL doesn't know UTC state; the APU stamps
it at decode time (see §4). PL contributes only cycle-lock provenance flags.

Note: after this, MTR1 has 2 spare words. Future 150/180-cycle aggregates
will need their own record format — that is the intended pattern
(registry keyed on word 1).

---

## 2. MSAP1_PL work items

All new RTL lives under `SourceData/DesignFile/` inside the MeterCore
hierarchy (AGENTS.md: single module-reference boundary; new logic must be a
pure observer of `frame_accept` — no `ready`, no backpressure).

1. **Widen the sample counter to 64-bit** (`adc_conversion.vhd`).
   Keep `TUSER[31:0]` as the low word (existing consumers untouched); place
   the high 32 bits in the currently-unused `TUSER[105:74]` window — the 54
   free bits mean **no TUSER width change**, no `xpm_fifo_axis` generic
   change. Free-running from reset, increments per accepted frame, never
   reset by configuration apply or capture restart (only `aresetn`).
   Leave `meter_frequency_estimator` untouched this milestone (it keeps
   using the low 32 bits + mod-2^48 math; removing that is a later cleanup).

2. **New `grid_cycle_timing` entity** (suggest
   `MeterProcessing/grid_cycle_timing.vhd` + `MeterCommon/grid_timing_pkg.vhd`).
   - Inputs: `frame_accept`, 64-bit sample counter (from TUSER), qualified
     crossing strobes, `reference_valid`, active grid config, apply toggle.
   - Reuses the **existing** `meter_zero_crossing` instance's outputs — do
     not build a second detector. Extend `meter_zero_crossing` with a
     falling-crossing output (new ports, existing behavior and rising-edge
     outputs unchanged) so `half_cycle_boundary` can be generated and
     tested; nothing consumes it yet.
   - Outputs: `cycle_boundary`, `half_cycle_boundary`, `cycle_sequence`,
     `basic_block_boundary`, `cycles_in_current_block`, lock/fallback flags,
     block-start sample index latch.
   - Block rule: count complete cycles (rising crossing → rising crossing);
     after `cycles_per_block` complete cycles, assert
     `basic_block_boundary` on that frame. Boundary convention: the frame
     carrying the closing crossing is the **first frame of block N+1**, so
     `first_sample(N+1) = last_sample(N) + 1` — gapless by construction.
   - Fallback: if `reference_valid` drops or no qualified crossing arrives
     within ~2 nominal cycles' worth of samples, close blocks by the
     fallback window (`SHADOW_WINDOW_SAMPLES`) and set `free_run_fallback`
     until crossings return; re-lock on the next qualified crossing. Blocks
     stay gapless across mode transitions.
   - On apply toggle: restart the block cleanly (flag
     `first_block_after_apply`), consistent with existing shadow/active
     commit discipline.

3. **`meter_rms` boundary + divisor changes** (`meter_rms.vhd:338-368`).
   - Window close driven by `basic_block_boundary` when cycle timing is
     enabled (sample-count comparison retained as the fallback path).
   - **Divide by the actual accumulated count** (`snapshot sample_count+1`),
     not `active_window_samples` — required for variable-length blocks and
     correct in both modes. This is the highest-risk metrology change;
     testbench golden values must cover it.

4. **Registers** — add the three `GRID_*` registers to
   `meter_processing_axi_regs.vhd` (constants in `grid_timing_pkg.vhd`,
   write arm, read arm, shadow signal, thread through `meter_core.vhd`).

5. **MTR1 v2 assembly** (`MeterResultHub_Wrapper.vhd`): format word, word 6
   = actual count, word 15 timing word, words 60–61 first-sample index
   (latched at block start by `grid_cycle_timing`).

6. **Waveform correlation alignment** (`meter_waveform.vhd`): latch the
   TUSER-carried 64-bit sample counter into the existing
   `frame_sequence` correlation registers instead of the module's local
   pre-FIFO counter. Kernel driver and ioctl ABI stay unchanged; the field
   now means "conversion-domain sample counter", which is exactly what the
   APU UTC mapping needs. Document the residual elasticity-FIFO offset
   bound (≤16 frames = 0.5 ms @ 32 kSPS) in the README.

7. **Testbenches + scripts**: new `grid_cycle_timing_tb.sv`; extend
   `meter_core_tb.sv` (`check_meter_word()` asserts every word — must be
   updated for v2) with a sine on CH6 so crossings fire; add every new file
   to the hardcoded lists in `check_meter_core.tcl`,
   `check_metering_synthesis.tcl`, `check_metering_module_references.tcl`.

8. **Docs**: update `MeterProcessing/README.md` (register + record
   contracts), `MeterCore/README.md` (correlation semantics), and PL
   `AGENTS.md` (new entity + registers — required by its own rules).

Verification: `check_meter_core.tcl`, `check_meter_frequency.tcl`,
`check_metering_pipeline.tcl`, focused synthesis, then
`implement_ad7771_design.tcl` → new XSA for the chain.

## 3. MSAP1_RPU work items

1. `common/include/rpu_control_protocol.h`: append `nominal_frequency_hz`
   (keep byte-identical with the APU copy); update static_assert 172→176.
2. `common/include/metering.hpp`: add `nominal_frequency_hz` (default 60)
   to `Configuration`.
3. `common/src/metering.cpp`:
   - `valid_configuration()`: accept only 50/60; keep existing checks.
   - Derive `cycles_per_block` (50→10, 60→12); add `GRID_*` offsets to the
     constant block; write shadow, apply, and **verify readback** of
     `GRID_ACTIVE_CONFIG` in the same `configure()` transaction (an
     unverified register would silently accept an unlatched value).
   - `status()`: read `GRID_STATUS` into `Status` (internal health only —
     RPMsg health payload unchanged this milestone; only 42 B headroom).
4. `R5c0/MainApp/handlers/meter/meter_config.cpp`: map the new payload
   field into `Configuration` within the existing single transaction.
5. Build `r5c0` and `r5c1` via `vitis -s scripts/build_r5_apps.py -- all`;
   recreate platform from the new XSA. Update RPU `AGENTS.md` (ABI) and
   `docs/AD7771.md`.

No MTR1 knowledge, no DMA, no zero-cross logic on RPU — unchanged split.

## 4. MSAP1_APU work items

1. **Types** (`common/msap1/meter/`, suggest new `meter_timing.hpp`):
   ```cpp
   enum class NominalFrequency : std::uint8_t { Hz50 = 50, Hz60 = 60 };
   enum class MeasurementPeriod : std::uint8_t { Basic, Cycles150_180, Min10, Hour2 };
   enum class TimeQuality : std::uint8_t { Unsynchronized, Synchronized, Holdover };
   ```
   **Replace `UpdatePeriod` outright** (rather than adding
   `UpdatePeriod::basic`): grep shows `MeterData` has no consumers, so the
   6-slot store shrinks to 4 cheaply. `duration(UpdatePeriod)` is removed;
   Basic has no fixed duration — actual duration comes from
   `sample_count / sample_rate` per block; a `nominal_duration()` helper
   may exist for display only.

2. **`BasicMeasurementBlock`** (decoder output, carried in `MeterUpdate`):
   sequence (u64, wrap-extended from wire u32), configuration_generation,
   first_sample_index, sample_count, cycle_count, nominal_frequency,
   cycle_locked/fallback flags, TimeQuality, optional UTC start timestamp,
   `FundamentalValues fundamental`. `MeterData` remains storage/publication
   only — no aggregation logic.

3. **Decoder**: register format `0x00010002` in `with_builtin_decoders()`
   producing `MeasurementPeriod::Basic` + the block above; keep the
   `0x00010001` decoder for stored v1 records. Delete the hardcoded
   `ms200` at `meter_data.cpp:316` as part of the v1 decoder's migration.

4. **Measurement timebase / UTC sync** (new
   `common/msap1/meter/measurement_timebase.{hpp,cpp}` in `msap1::meter`):
   - `TimeSyncPoint { uint64_t sample_counter; uint64_t utc_ns; uint64_t uncertainty_ns; }`
   - State machine: startup → `Unsynchronized`; valid sync →
     `Synchronized`; sync older than a staleness threshold (e.g. 3× the
     resync interval) → `Holdover`; new sync → `Synchronized`.
   - `utc_for_sample(index)` = linear mapping from the latest sync point.
     UTC steps update the *mapping* only — never the counter.
   - Sync source: the existing waveform correlation ioctl (bracketed
     CLOCK_TAI/REALTIME reads around an atomic PL latch), now returning the
     conversion sample counter (PL item 6). `CaptureCoordinator` refreshes
     the sync point periodically (e.g. every 10 s).
   - Timing quality never marks the electrical measurement invalid —
     `MeasurementQuality` and `TimeQuality` stay separate fields.

5. **Ingestor** (`record_ingestor.cpp:59-99`): replace the
   `window_samples == wire.rms_window_samples` equality check (wrong for
   variable blocks) with generation check + **sample-range continuity**
   (`first_sample(N+1) == first_sample(N) + sample_count(N)`) + sequence
   continuity. Gap ⇒ count `invalid_records` / log, same as today.

6. **Settings + wire encode**: `MeteringSettings.nominal_frequency_hz`
   (validate 50/60), `config/settings/factory-defaults.json`,
   `to_meter_configuration()` / `MeterConversionFile` (absent field
   defaults to 60 — decide whether to bump the conversion-file schema),
   `prepare_meter_configuration()`: set wire field, derive
   `rms_window_samples = sample_rate * cycles / nominal` (fallback window),
   stop deriving from `window_ms`.

7. **Web backend**: settings routes need no code (glaze reflection); extend
   `GET /api/v1/meter/readings` rendering in `meter_routes.cpp` with the v2
   timing fields (block sequence, first_sample_index, cycle_count, nominal,
   cycle_locked, time_quality) for on-target validation.

8. **Docs**: new `docs/TIMING_MODEL.md` (see §7) + `AGENTS.md` ABI notes.

## 5. MSAP1_WEB work items

Small, self-contained:

1. `src/api.ts`: `nominal_frequency_hz: 50 | 60` on
   `ProductSettings['metering']`; add v2 timing fields to `MeterReadings`
   types as exposed by item 4.7.
2. `src/App.tsx` Configuration → Meter tab: a `<select>` (50 Hz / 60 Hz)
   following the existing frequency-mode select pattern, saved through the
   existing `saveSettings()` whole-document flow. Label it "Nominal grid
   frequency" and note "Basic block: 10 cycles" / "12 cycles" dynamically.
3. Replace the hardcoded dashboard strings "Mean-corrected 200 ms RMS" /
   "Update period: 200 ms" (`App.tsx:1024,1028`) with cycle-based wording
   driven by the readings payload.
4. Update `README.md`; verify with `npm ci && npm run build` (no test
   framework exists in this repo).

---

## 6. Test plan (maps milestone §13–14)

**PL testbenches** (deterministic, no hardware):
- 50 Hz: exactly 10 cycles per block; 60 Hz: exactly 12.
- Off-nominal (49.9/50.1, 59.9/60.1 equivalents scaled to TB rates): blocks
  still close on complete cycles; block sample counts vary; **no fixed
  sample count assumed**.
- Gapless: `first_sample(N+1) == first_sample(N) + count(N)` across many
  blocks, including across a lock→fallback→relock transition.
- Sample counter strictly monotonic; unaffected by config apply.
- Config apply mid-stream: clean block restart, `first_block_after_apply`.
- MTR1 v2 word-exact check in `meter_core_tb`.

**APU unit tests** (plain `main()` style, `common/CMakeLists.txt`):
- `protocol_test`: nominal field encode; 50/60 accepted; 55 rejected;
  generation fingerprint changes with nominal.
- `meter_data_test`: v2 fixture (extend `periodic_record()`); decodes to
  `MeasurementPeriod::Basic` with correct timing metadata; v1 record still
  decodes.
- New `measurement_timebase_test`: Unsynchronized → Synchronized →
  Holdover → Synchronized; UTC step changes mapping but not counter;
  `utc_for_sample` monotonic between syncs.
- `acquisition_architecture_test`: ingestor continuity/gap detection,
  u32 sequence wrap.
- `settings_test`: 50/60 accepted, others rejected; default present.
- Note: build is aarch64 cross — ctest runs per the team's target flow,
  not on the x86_64 host.

**Simulator-driven target validation**: the existing PL simulator + settings
path already supports 49.9…60.1 Hz via `simulator_frequency_millihz`; extend
`tests/method/test_adc_rpmsg_procedure.md` with: nominal=50 and 60 checks of
cycle_count, block sequence increment, first-sample continuity, and
time-quality transitions (stop `msap1-fpga-acquisition` sync refresh →
Holdover).

## 7. Documentation deliverable

`MSAP1_APU/docs/TIMING_MODEL.md` (cross-referenced from PL/RPU docs):
- A BasicMeasurementBlock is **cycle-defined, not time-defined**; 50 Hz → 10
  cycles, 60 Hz → 12 cycles; nominal ≈ 200 ms but 200 ms is never the
  semantic definition, and actual duration varies with grid frequency (intentional).
- Two time domains: measurement time (PL 64-bit sample counter, never
  stepped) vs UTC (mapping via sync points, corrected independently).
- Nominal vs measured frequency vs cycle timing — three separate concepts.
- Ownership: PL = cycle timing + monotonic counter; RPU = configuration/
  control conduit; APU = UTC sync authority + decoded data.

## 8. Sequencing / merge order

The product ships as one Yocto image, so the repos can move in lockstep,
but build/test order matters:

1. **Contract sign-off** (this document §1) — record v2 layout, register
   map, wire field.
2. **PL** — RTL + testbenches green → `implement_ad7771_design.tcl` → new
   XSA (`runtime-generated/bin_file/MSAP1_PL.xsa`). No address-map change,
   so no device-tree/mconf churn beyond the normal chain.
3. **RPU** — protocol header + driver; platform regen from new XSA; build
   r5c0/r5c1.
4. **APU** — settings, protocol (byte-identical header), decoder v2,
   timebase, ingestor, web-backend; unit tests.
5. **WEB** — types + selector + strings; `npm run build`.
6. **Integration** — `./make_PL.sh && ./make_mconf.sh && ./make_RPU.sh &&
   ./make_yocto.sh`; run the extended target procedure at 50 and 60 Hz on
   the simulator source.
7. Per workspace rules: separate commits per repo, no combined branches,
   no commit/push without explicit request.

## 9. Risks & open decisions

| Item | Recommendation |
|---|---|
| RMS divisor change (configured → actual count) subtly changes results | Cover with TB golden values in both fixed and cycle modes; correct in both |
| Variable block length breaks ingestor window equality check | Replaced by continuity checks (§4.5) — must land with the PL change |
| Only 2 MTR1 words remain free after v2 | Accepted; future aggregates get their own record formats via the registry |
| Dead/invalid reference channel | Free-run fallback keeps blocks gapless; flagged, never silently |
| UTC sync accuracy bounded by elasticity-FIFO offset (≤0.5 ms) | Document; well within this milestone's needs; refine in the aggregation milestone |
| Factory default nominal frequency | Propose 60 Hz — **confirm per target market** |
| `rms.window_ms` setting | Keep in schema, mark superseded; remove in a later schema bump |
| `UpdatePeriod` full replacement vs additive | Full replacement (no consumers exist); if review disagrees, fall back to adding `basic` and migrating only the decoder |
| Conversion-file schema bump (v3 → v4) for the new field | Prefer default-when-absent in v3 to keep the merge window small |
| TimeSyncPoint clock (CLOCK_REALTIME vs TAI) | Reuse the waveform TAI-bracket pattern, store both; expose UTC |

## 10. Explicit non-goals honored

No 150/180-cycle, 10-min, or 2-h aggregation; no harmonics, flicker,
dip/swell/interruption/RVC; no PTP/GPS/PLL discipline; no energy/demand/power
changes; no unrelated refactoring. The frequency estimator implementation is
preserved untouched; `grid_cycle_timing` only consumes the existing
zero-cross detector. `half_cycle_boundary` is generated and tested but feeds
nothing yet.
