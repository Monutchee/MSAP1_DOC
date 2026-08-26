# M15 UTC-overlap target validation — 2026-08-26

## Target and restored state

- Device: `172.30.19.99` (`msap1`), coordinated PL/RPU/APU/Yocto deployment.
- Source: isolated PL simulator, 128,000 frames/s.
- Original and final settings: nominal 60 Hz, simulator 60 Hz, content hash
  `16f02c09405cf3c5b650fb169e7d017aeee25d2a1aee36977ac2222d4de16cfb`.
- Final health after recovery: PASS; generation `0x5c631962` matches PL;
  capture active; 133,641/133,641 R5C1 inputs valid; 51,960/51,960 R5C1
  records emitted; zero R5C1 CRC, format, length, FIFO, ring, continuity,
  output-error, or output-drop counts; zero APU DMA read errors, invalid
  records, sequence gaps, DMA overruns, FIFO overflows, and header errors.

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
[m15_target_overlap_20260826.txt](../../../../runtime-generated/testLog/m15_target_overlap_20260826.txt).

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
[m15_target_overlap_50hz_20260826.txt](../../../../runtime-generated/testLog/m15_target_overlap_50hz_20260826.txt).

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
document was restored.

## Verdict and remaining scope

The deployed UTC Basic and 150/180-cycle overlap implementation passes one
arbitrary-phase target boundary at both 50 Hz and 60 Hz. This closes the
specific overlap-evidence gate. It does not close the separate arbitrary-phase
matrix, fault-injection/invalidation matrix, multi-boundary soak, clean two-hour
result, full PL timing/CDC/DRC evidence, or formal IEC conformance gates.
