# M15 target and remaining-gate validation — 2026-08-26

## Scope and overall result

This report began as the 50/60 Hz UTC-overlap capture and was expanded after
the remaining M15 host, PL, APU, build-report, and available target lifecycle
checks were run. All runnable host/build checks passed. The deployed target
also passed UTC overlap at both nominal frequencies, a clean ten-minute
result, clean live two-hour arithmetic, lock-loss recovery, and transactional
source-change failure.

M15 remains formally open for the post-deploy UTC startup-phase matrix, APU
restart/remapping check, first completed clean two-hour result, and external
IEC 62586-2 functional/uncertainty qualification. These open checks do not
invalidate the independent M16 spectral architecture work.

## Target and restored state

- Device: `172.30.19.99` (`msap1`), coordinated PL/RPU/APU/Yocto deployment;
  target build hash `153f29`, build time `2026-08-26 11:45:22 UTC`.
- Source: isolated PL simulator, 128,000 frames/s.
- Original and post-test settings before the lifecycle reboot: nominal 60 Hz,
  simulator 60 Hz, content hash
  `16f02c09405cf3c5b650fb169e7d017aeee25d2a1aee36977ac2222d4de16cfb`.
- Health after overlap-probe recovery: PASS; generation `0x5c631962` matches PL;
  capture active; 133,641/133,641 R5C1 inputs valid; 51,960/51,960 R5C1
  records emitted; zero R5C1 CRC, format, length, FIFO, ring, continuity,
  output-error, or output-drop counts; zero APU DMA read errors, invalid
  records, sequence gaps, DMA overruns, FIFO overflows, and header errors.

The later lifecycle reset intentionally issued a Linux reboot. Because this
image was loaded through volatile JTAG rather than persistent boot media, the
target did not autonomously return. A coordinated `mnc deploy` is required
before the remaining on-device checks can continue; the reset outcome is a
deployment-property finding, not an aggregation arithmetic failure.

## 60 Hz / 180-cycle boundary (13:10 UTC)

- Pre-boundary Basic 6586: samples 168,583,831 through 168,609,430,
  25,600 samples, timing `0x00010c3c`.
- UTC-resynchronized Basic 6587: samples 168,598,764 through 168,624,363,
  25,600 samples, timing `0x00090c3c` (bit 19 set).
- Basic overlap is 10,667 samples. The marked range advances past the old
  range; it is neither an unmarked overlap nor a contained duplicate.
- Continuing aggregate 439: Basic sequences 6573..6587 (15), 180 cycles,
  status `0x0000000e` (complete, frequency valid, UTC overlap), 384,000
  contributed samples, samples 168,251,031 through 168,624,363, physical span
  373,333. Contribution count exceeds physical span by exactly 10,667.
- Synchronized aggregate 440: Basic sequences 6587..6601 (15), 180 cycles,
  status `0x00000016` (complete, frequency valid, UTC resynchronized), 384,000
  contributed samples and a 384,000-sample physical span.
- Both records report aggregation continuity count 0. APU health remained at
  zero invalid records and zero gaps through the transition.

Raw record output:
[m15_target_overlap_20260826.txt](evidence/m15_target_overlap_20260826.txt).

## 50 Hz / 150-cycle boundary (13:20 UTC)

- Applied only the nominal-frequency and simulator-frequency fields (60→50),
  producing settings hash
  `8f6f2626b6504644b7fb340155768bcb31949a7d3e68862ff5fbacb7f2f10531`.
- Before the boundary, the authenticated API reported a locked 10-cycle Basic
  at 50.000 Hz and a 15-Basic / 150-cycle aggregate.
- Pre-boundary Basic 9586: samples 245,387,247 through 245,412,846,
  25,600 samples, timing `0x00010a32`.
- UTC-resynchronized Basic 9587: samples 245,397,487 through 245,423,085,
  25,599 samples, timing `0x00090a32` (bit 19 set).
- Basic overlap is 15,360 samples and the marked range advances.
- Continuing aggregate 639: Basic sequences 9583..9597 (15), 150 cycles,
  status `0x0000000e`, 384,000 contributed samples, samples 245,310,447 through
  245,679,086, physical span 368,640. Contribution count exceeds physical span
  by exactly 15,360.
- Synchronized aggregate 640: Basic sequences 9587..9601 (15), 150 cycles,
  status `0x00000016`, 384,000 contributed samples and a 384,000-sample
  physical span.
- Both records report aggregation continuity count 0.

Raw record output:
[m15_target_overlap_50hz_20260826.txt](evidence/m15_target_overlap_50hz_20260826.txt).

## Diagnostic-probe correction

The first recovered probe registered its disposable stream consumer at cursor
zero and replayed roughly 45,000 retained records. That replay was intrusive:
after the required 50 Hz boundary evidence had already been captured, it
starved the acquisition publication path and induced kernel DMA overruns,
sequence gaps, and 79 rejected records. The timestamps begin during the probe
and precede the 50→60 settings restoration, so they are not an APPLY-transition
failure.

The probe was corrected to query the stream head and acknowledge that cursor
before starting its timed capture. A five-second regression of the corrected
probe captured 98 newly produced records and left health PASS with no warning
entries, overruns, invalid records, or gaps. Acquisition was restarted once to
clear the test-induced cumulative counters, and the exact original settings
document was restored. The post-recovery health snapshot is retained in
[m15_target_final_health_20260826.txt](evidence/m15_target_final_health_20260826.txt).

## Host, PL, APU, and routed-build evidence

- The production R5C1 exact-golden/reference suite passed with the opt-in M15
  invalidation matrix enabled. Its randomized soak covered 718 Basic blocks
  and 38 interval closures. The matrix checked 28 ten-minute results across
  startup, sequence gaps, lock/fallback changes, APPLY, source/generation,
  reset/remap, UTC correction, and nominal-frequency changes: each first
  recoverable interval was contaminated and each following complete interval
  was clean.
- Preview-enabled and preview-disabled reference runs each emitted 4,000
  completed records with digest `ac8d436cda19f9f5`; completed serialized
  records compared byte-for-byte.
- The R5C1 transport/service suite passed valid input, malformed input, ring
  pressure, sequence wrap/repeat/gap/out-of-order, output retry, and fail-closed
  cases.
- PL focused verification passed the SingleCycle exporter whole-packet
  overflow/recovery test, complete metering-pipeline simulations, MeterCore,
  and simulator-disabled production-shape elaboration.
- The unrestricted APU suite passed 25/25 tests after the reusable target probe
  was added and built.
- The coordinated 2026-08-26 HLS/PL/RPU/mconf/Yocto build passed. Routed K26
  evidence reports WNS `+0.528 ns`, TNS `0`, 0 unsafe/unknown CDC endpoints,
  no effective congestion above level 5, and no DRC errors or critical
  warnings. Utilization is 44,528 LUT, 62,838 FF, 82 BRAM, 209 DSP, and
  9,236/14,640 physical CLBs (63.09%).

Reproduction commands, from each component repository unless noted:

```sh
MNC_REQUIRE_M15_INVALIDATION_MATRIX=1 \
  bash R5c1/tests/run_aggregation_engine_reference_tests.sh
bash R5c1/tests/run_aggregation_shadow_tests.sh

vivado -mode batch \
  -source SourceData/Script/AI_gen/check_r5_aggregation_export.tcl
vivado -mode batch \
  -source SourceData/Script/AI_gen/check_metering_pipeline.tcl
vivado -mode batch -source SourceData/Script/AI_gen/check_meter_core.tcl

ctest --test-dir build --output-on-failure
./mnc PL summary
```

The coordinated build transcript is
`runtime-generated/buildLog/build_20260826_072505.log` at the workspace root.

## Additional target evidence

### Clean ten-minute result

The authenticated API returned completed ten-minute sequence 7 at generation
`0x5c631962`: 76,799,996 samples, 36,000 cycles, nominal 60 Hz, 3,000 source
intervals, valid 120 V / 3 A lanes, `arithmetic_error=false`, time aligned,
uncontaminated, and boundary valid.

### Live two-hour arithmetic

Before a completed two-hour result existed, the API returned live two-hour
sequence 3 with 153,599,993 samples, 72,000 cycles, and two clean source
intervals. It was correctly marked open and non-normative while retaining
valid 120 V / 3 A lanes, `arithmetic_error=false`, aligned timing, no
contamination, and valid boundary provenance. This verifies the widened
long-window arithmetic on target but does not replace the required completed
12-interval result.

### Lock loss and source transaction

- A phase-continuous three-phase simulator interruption changed the grid status
  from locked to fallback/unlocked and back to locked. R5C1 input/output,
  continuity, DMA, invalid-record, gap, FIFO, and header counters remained
  clean after recovery.
- A requested switch to the unavailable physical ADC failed closed during the
  coordinated service transaction. Health recovered and the configured source
  remained the simulator; an explicit simulator selection then succeeded. The
  persistent settings hash remained unchanged.

### Reset/deployment lifecycle

At `2026-08-26 14:01:39 UTC`, a coordinated reset check issued `sync` followed
by Linux reboot. SSH dropped as expected, but the JTAG-loaded volatile image
did not reboot itself; the neighbor entry subsequently failed and SSH reported
`No route to host`. The obsolete expectation of autonomous Linux reboot
recovery has therefore been replaced by a coordinated deploy-and-health gate.

## Reusable target test recovery

The corrected probe is valuable as a live integration test because it observes
the production durable meter stream and verifies the exact Basic and
150/180-cycle boundary geometry. It has been recovered into the APU repository
as:

- `tests/target/m15_utc_overlap_target_test.cpp`
- `tests/target/README.md`

The CMake target is `m15_utc_overlap_target_test`. It is intentionally excluded
from normal builds and CTest because it requires a live target and an exact UTC
boundary. The test uses the production typed IPC client, acknowledges the
current stream head before reading, validates both overlap branches, and exits
nonzero for missing evidence or any sequence/sample/continuity mismatch. The
source builds cleanly on the host; a reusable target run must use the AArch64
APU build.

The recovered architecture-specific binary was not added to Git, and the
Python copy was not retained because it duplicated the private IPC wire
decoder and would drift independently from the production typed client.

## Verdict and remaining scope

The deployed UTC Basic and 150/180-cycle overlap implementation passes one
arbitrary-phase target boundary at both 50 Hz and 60 Hz. This closes the
specific overlap-evidence gate. The deterministic host invalidation matrix,
available target fault cases, complete APU suite, full coordinated build, and
routed timing/CDC/DRC/utilization reviews also pass.

Remaining on-device work after redeployment is:

1. exercise 50 Hz and 60 Hz startup immediately before, on, and after a UTC
   ten-minute boundary;
2. restart APU acquisition and verify UTC remapping/recovery;
3. retain a clean multi-boundary run through exactly 12 consecutive ten-minute
   intervals and validate the completed 921,600,000-sample two-hour record.

Formal IEC 62586-2 functional and uncertainty qualification remains a valid
external laboratory prerequisite for a product-level Class A claim; it is not
an obsolete PL-aggregation test and cannot be closed by the simulator campaign.
