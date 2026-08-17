# ADC control-SPI corruption — root cause found, not fixable in current hardware

Device msap1(ip: xxx.xxx.xxx.xxx) boot 2026-08-17 ~14:40 UTC, bitstream md5
`1ac891f9a6e830686dc46749fe238590` (MSAP1_PL `c1bf5f4`, RPU `9dcd385`,
APU `a22f735`). AD7771 on a ~200 mm unshielded flying lead to the KR260
Raspberry Pi header.

**Verdict: the AD7771's own DCLK/DOUT conversion-data interface is the
aggressor corrupting the control SPI, coupling through the shared cable.
Confirmed by controlled experiment at p ≈ 1e-9. It cannot be fixed in
firmware or fabric — the sensor board exposes DCLK and SPI on the same
socket, so the coupling path is not modifiable. Closed as a documented
limitation, mitigated in software.**

Nothing measured here affects published measurement values. The corruption
is confined to the control/status SPI path.

## Files

- `ab-test-dclk-aggressor.txt` — raw output of the decisive A/B experiment.

## Symptom as originally reported

Chronic, low-rate malformed replies on AD7771 register reads, surfacing as
intermittent "ADC degraded" health reports with no accompanying measurement
fault. DRDY never faltered; the conversion data path stayed clean throughout.

Baseline before any change: **~8.8 malformed reply headers per hour**
(89 in 10 h 29 m, plus 22 in 2 h 04 m across a firmware change, at SCK
6.25 MHz). Every one was recovered by the existing 3-attempt retry — the
protocol-error and retry-recovery counters were always equal, so no read
ever failed permanently.

## The decisive experiment

The AD7771's DOUT interface in 4-lane format clocks 64 bits per lane per
frame at a fixed DCLK of 8.192 MHz. At 128 kSPS that is 8.192 Mbit/s per
lane — the interface is **saturated, ~100 % duty**. Lowering the output data
rate leaves DCLK's frequency at 8.192 MHz but makes it bursty, cutting the
aggressor's duty cycle without touching cable, SCK, or any register.

200 forced health sweeps per arm (`mnc meter health --refresh`, 97 register
reads each), 112 s per arm, equal exposure:

| metric | 128 kSPS (~100 % duty) | 1 kSPS (~0.8 % duty) | ratio |
|---|---|---|---|
| SPI protocol errors | 13 | 1 | **13×** |
| `GEN_ERR_REG_1` events | 24 | 1 | **24×** |
| Config-read mismatches | 2 | 0 | — |

Poisson probability of observing ≤1 event in arm B if the rate were
unchanged: **3.2e-5** (protocol errors), **9.4e-10** (`GEN_ERR_REG_1`
events). The aggressor is identified beyond reasonable doubt.

## Mechanism

The ADC raises **`SPI_INVALID_READ_ERR`** (`GEN_ERR_REG_1` bit 3), meaning it
decoded an *invalid register address* off SDI. So the corruption is on the
**outbound** path — DCLK edges coupling into SDI/SCLK in the shared bundle —
not on the reply path.

That accounts for every observation:

- **All malformed headers land in histogram bucket `0x0_`** (22 of 22).
  An invalid address means the ADC never drives a valid `0010 0000` header,
  so the top nibble is always zero. Systematic, not random — which is what
  the per-nibble histogram was added to distinguish.
- **The DOUT path itself is clean** (`header_error_count` 0). It is the
  aggressor, not a victim: the SPI transacts roughly 0.001 % of the time
  (248 µs of wire activity per 30 s audit), so DOUT is essentially never
  aggressed back. The asymmetry is exposure, not robustness.
- **SDO drive strength is irrelevant** — wrong direction entirely.
- **Halving SCK changed nothing** — it reduces neither the aggressor's duty
  cycle nor the 16 SCLK edges per transaction.

## Hypotheses falsified along the way

Recorded so they are not re-investigated.

| Hypothesis | Falsified by |
|---|---|
| SCK too fast | 6.25 MHz against the AD7771's 30 MHz maximum (t12). Not close. |
| Missing inter-transaction delay | t19 minimum CS-high time is **10 ns**; back-to-back `XSpi_Transfer` calls already give hundreds. |
| Cable cannot carry ~6 MHz | DCLK carries **8.192 MHz** cleanly on the same bundle. Bandwidth is not the limit. |
| SDO drive strength | `SDO_DRIVE_STR` resets to `0x1` (strong), so "set strong" was a no-op — `CONFIG_2` read `0x49` before and after. Also the wrong direction. |
| Lowering SCK 6.25 → 3.125 MHz | Measured: 7.5 errors/h vs 8.8 baseline. No effect. |
| Concurrent SPI access on the R5 | Only `comm_task` reaches the ADC; `led_task` is heartbeat-only; R5c1 never opens the peripheral. |
| SDO hijacked by SAR / Σ-Δ readback mode | `CONFIG_2` bit 5 and `CONFIG_3` bit 4 both written 0 and verified every poll. |
| Pin-level SCK↔SDI timing skew | 320 ns bit period against a 5 ns SDI setup/hold requirement (t20/t21) — ~60× margin. |

## Why it cannot be fixed

The sensor board exposes DCLK/DOUT and the control SPI **on the same
socket**, so the two groups cannot be separated, given ground returns, or
individually damped without a board respin. The three physical fixes that
would work are therefore unavailable:

1. Route SPI in a separate bundle from DCLK/DOUT.
2. Interleave ground returns adjacent to SCK and MOSI.
3. Series damping resistors (33–100 Ω) at the FPGA end of SCK/MOSI/SS.

There is also **no configuration headroom**: at 128 kSPS the data interface
is saturated (8 channels × 32 bits over 4 lanes at 8.192 MHz = 100 % duty),
so the aggressor's duty cycle cannot be reduced at the operating rate.

The one untried register-level mitigation is `DOUT_DRIVE_STR`
(`GENERAL_USER_CONFIG_2` bits[2:1], currently reset/nominal, `10` = weak),
which lowers the aggressor's edge amplitude rather than its duty. Untested;
it perturbs a data path that currently runs clean, and `header_error_count`
is the only visibility into whether it degrades.

## Mitigations in place

The corruption is now non-damaging and observable rather than silent.

**RPU** (`9dcd385` and predecessors, PR #18)

- 3-attempt retry with 10 µs spacing on every register read. Recovers 100 %
  of malformed-header cases; protocol-error and retry-recovery counts have
  been equal across every observation window.
- Confirming (read-twice-compare) read on the **17 R/W registers behind
  `configuration_matches`** — `0x00`–`0x07`, `0x11`–`0x14`, `0x60`–`0x64`.
  This catches the corruption class the header check structurally cannot
  see: a bad byte in the *data* half of the reply leaves a valid header.
  **Scoped deliberately.** Applying it to the sweep's clear-on-read
  registers (`0x4C`–`0x53`, `0x54`–`0x57`, `0x59`, `0x5B`, `0x5D`–`0x5F`)
  reports every ADC error bit as `0x00` by construction, because the first
  read clears the latch and the second returns zero. That regression shipped
  once (`edab967`) and was reverted.
- `GEN_ERR_REG_1` retained in software (sticky OR + event count). The
  register is clear-on-read and the sweep reads it every poll, so before
  this anything that fired between polls was consumed and lost.
- Malformed reply headers bucketed by high nibble, so a systematic
  corruption is distinguishable from random mis-sampling.

**APU** (`a22f735`, PR #46)

- A lost `INIT_COMPLETE` / `CONFIG_MATCH` on an otherwise-successful sweep
  now requires the next audit to agree before publishing. Previously the
  "single bad audit must not replace a known-good cache" debounce only ran
  when the *transport* failed, so a corrupted data byte in a successful
  sweep published straight to the cache — one bad byte was enough to report
  a degraded ADC that was never degraded. **This was the actual cause of the
  false health warnings.**

**PL** (`c1bf5f4`, PR #29)

- SCK 6.25 → 3.125 MHz (`Multiples16 = 2`, generated `C_SCK_RATIO = 32`).
  No measured benefit; retained because the interface carries configuration
  traffic only and the extra margin is free.
- Explicit pin drive: MOSI/SS `DRIVE 8 SLEW SLOW`; SCK held at `DRIVE 12`
  deliberately — a slow clock edge lingers near the ADC's input threshold
  and a glitch there reads as an *extra clock*, desynchronising the frame.

## Residual risk

**Writes are the highest-consequence exposure.** `write_adc_register()` has
no retry, and now that addresses are known to corrupt, a write could in
principle land in the wrong register. Mitigating factors: configuration runs
once at boot, `update_adc_register()` is read-modify-write, and
`configuration_matches` re-verifies the outcome every 30 s. **Gap:**
registers outside the verify mask — channel offset/gain calibration
(`0x1C`–`0x4B`), sync offsets (`0x09`–`0x10`), mux config, GPIO, buffer
config — would not be caught. A silent calibration corruption would present
as a wrong measurement, not as a health fault. `SPI_INVALID_WRITE_ERR`
(`GEN_ERR_REG_1` bit 2) appearing in the sticky latch is the signal to
investigate this; it has not appeared to date.

**The conversion data path has no CRC**, so a bit error in Σ-Δ payload is
undetectable — only the per-channel header is checked. Reasoning suggests it
is genuinely much cleaner rather than merely unobserved: at the SPI's
per-bit error rate, 32.8 Mbit/s of DOUT traffic would produce on the order
of 1,800 bit errors per second and obvious measurement noise. None is seen,
consistent with DOUT being the aggressor rather than the victim.

**`ROM_CRC_ERR`** (`GEN_ERR_REG_1` bit 5) is present in the sticky latch.
The datasheet says a device reset is required if it genuinely triggers.
However the sticky value is itself accumulated over the corrupting bus, and
an address-corrupted read of `0x59` could fold in another register's
contents as phantom bits. Unresolved. Per-bit counters would separate a
one-shot power-up event from a recurring fault; the current OR conflates
them.

**`GEN_ERR_REG_2`** (`0x5B`) has no sticky latch, so `RESET_DETECTED` and
`EXT_MCLK_SWITCH_ERR` are still consumed and discarded on every poll.

**`RpuHealthMonitor` has no automated coverage.** The debounce branch was
verified by inspection across the transient, genuine-loss, recovery and
cold-start cases. Adding a test needs `RpuController` behind an interface
first.

## How to re-measure

```sh
# Current state — all counters are cumulative since RPU start
mnc meter health --full | grep -iE \
  "SPI protocol|retry recover|Config mismatch|GEN_ERR_REG_1 events|latched SPI|Bad header buckets"

# Force one real 97-read sweep (the plain health path returns the APU cache)
mnc meter health --refresh

# Reproduce the A/B result: ~0.065 protocol errors and ~0.12 GEN_ERR events
# per sweep at 128 kSPS, roughly 20x lower at 1 kSPS
mnc adc rate --sps 1000     # reduce aggressor duty
mnc adc rate --sps 128000   # restore
```

`GEN_ERR_REG_1` event count is the most sensitive instrument — it showed
24 vs 1 where protocol errors showed 13 vs 1, and it reflects the ADC's own
view of the fault rather than the master's.

## Revisit if

- **The sensor board is respun** — separate SPI from DCLK/DOUT, interleave
  ground returns, add series damping at the FPGA end. This is the real fix
  and the only one that removes the coupling.
- **`SPI_INVALID_WRITE_ERR` appears** in the sticky latch — writes reaching
  the wrong register is the one failure mode that could corrupt measurement
  data rather than just health reporting.
- **Config-mismatch rate climbs materially** above ~3–4/hour, indicating the
  coupling has worsened (cable re-seated, re-routed, or degraded).
- **`header_error_count` becomes non-zero** — the data path starting to
  suffer would invalidate the aggressor/victim asymmetry this analysis rests
  on.

## Unrelated finding recorded here

The DMA transport `callbacks` deficit sits static at **64–66** against a ring
depth of 64, and did not move across 60 aggregation windows (960 records) of
observation. It is a fixed startup offset — the DMA fills the ring while the
reader attaches, and those completions are covered by fewer interrupts — not
the phase-locked per-window interrupt coalescing that caused the
2026-08-13/14 sequence-gap incident. `produced == consumed` exactly, with
`gaps 0, overruns 0`. Nothing in the transport depends on the callback count
since the marker rewrite; see `2026-08-14/ANALYSIS.md`.
