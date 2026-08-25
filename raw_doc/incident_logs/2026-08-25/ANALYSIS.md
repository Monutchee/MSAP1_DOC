# R5 cores silently running at ~16.7 MHz instead of 533 MHz — Linux suspends the unclaimed RPLL

Device msap1 (ip: xxx.xxx.xxx.xxx), observed 2026-08-25 on the running
system via SSH + `devmem2`. Reads only; nothing on the device was modified.
Found while answering the simple question "does the R5 run at 533 MHz?" —
it does not, and by the mechanism involved it likely never has for more than
a few seconds per boot.

**Verdict: the RPU is configured to run from RPLL at 533.33 MHz and the boot
firmware does bring that up, but nothing in the Linux device tree references
the RPU's clock. The kernel's late-init unused-clock sweep therefore tells
PMU firmware to suspend RPLL — reset + bypass — out from under the running
R5 cores. A bypassed PLL passes the raw 33.333 MHz `ps_ref_clk` through, so
after every Linux boot the R5s execute at `33.333 / 2 ≈ 16.7 MHz`, about 3 %
of intended speed. This is a known ZynqMP-family trap, not a design error in
the Vivado clock configuration, which is correct as exported. Fix staged
(one kernel command-line flag via Yocto); not yet built, deployed, or
verified on hardware.**

The R5s keep running — slowly. Nothing crashes, no health fault is raised,
and `remoteproc` reports `running` for both cores, which is why this went
unnoticed. Physical metering was unaffected in its data path (the PL owns
sampling and DSP; fabric clocks come from IOPLL, which stays locked), but
every R5-side timing or performance observation made to date was taken at
~16.7 MHz and should be treated as invalid.

## Files

- `device-register-snapshot.txt` — the raw `devmem2` reads, remoteproc
  states, and `clk_summary` excerpt, with bit-field decode.

## Evidence

| Register | Read | Meaning |
|---|---|---|
| `CPU_R5_CTRL` (0xFF5E0090) | `0x03000200` | clock active, SRCSEL=0 (RPLL), divisor 2 |
| `RPLL_CTRL` (0xFF5E0030) | `0x00012D09` | FBDIV=45, DIV2 — but **RESET=1, BYPASS=1** |
| `PLL_STATUS` (0xFF5E0040) | `0x00000019` | IOPLL locked; **RPLL lock bit = 0** |

Corroborating, from the running kernel: `clk_summary` shows `rpll` with an
enable count of 0 (`rpll_int` disabled), and **no `cpu_r5` clock is modeled
in the Linux clock tree at all** — the kernel has no idea the RPU is a
consumer. Both remoteprocs report `running`.

The chain: `clk_disable_unused` runs at kernel late-init → the zynqmp clock
driver sees RPLL with zero enabled consumers → issues the disable through
the firmware interface → PMU firmware bypasses and resets the PLL. The R5s,
hard-wired to that mux, drop to the bypass clock mid-execution.

## Secondary finding: the deployed boot firmware is stale

The live `RPLL_CTRL` has FBDIV=45 (RPLL = 750 MHz → R5 = 375 MHz at boot,
before Linux kills it). The current hardware handoff in the repo
(`vivado_SDT_out/psu_init.c`) writes FBDIV=64 (RPLL = 1066.67 MHz → R5 =
533.33 MHz), matching the Vivado clock page. So the BOOT.BIN on this unit
was built from an older hardware description: even at boot this unit's R5s
ran at 375 MHz, not 533. The command-line fix alone would pin them at 375 —
a matched-artifact redeploy (FSBL/PMUFW/bitstream/RPU/APU) is required with
it, consistent with the rule from the 2026-08 RPU/PL version-coupling
incident.

## Impact assessment

- **Measurement data path: unaffected.** Sampling, DSP, and DMA live in the
  PL on IOPLL-derived fabric clocks. The AXI SPI's SCK is fabric-clocked, so
  ADC communication timing was also unaffected — only the R5's execution
  speed between transactions changed.
- **R5 workloads: everything ran ~32x slower than intended.** AdcController
  health sweeps, retry timing loops, and FreeRTOS tick-relative behavior all
  executed at ~16.7 MHz. Notably, all measurements in the 2026-08-17 SPI
  analysis (sweep durations, per-sweep rates) were taken in this state; the
  A/B conclusion stands (both arms equally affected) but absolute timings do
  not transfer to a 533 MHz R5.
- **Possible relevance to open incidents (speculative).** An RPU at 3 %
  speed is a plausible contributor to timing-sensitive anomalies such as the
  2026-08-14 #2 record-emission fault episodes. No evidence either way yet;
  the PL arbiter/packetizer remains the prime suspect there. Worth
  re-observing that fault after this fix is deployed, since it changes the
  RPU's real-time margins completely.

## Considered and rejected explanations

| Hypothesis | Rejected because |
|---|---|
| Vivado clock config wrong | PS config page and exported psu_init both specify RPLL/2 = 533.33 MHz correctly. |
| FSBL failed to program the PLL | FBDIV reads 45, not the silicon reset default (44) — software did program it; a *different* (older) psu_init value, and Linux later suspended it. |
| R5s halted/crashed | Both remoteprocs `running`; RPU firmware demonstrably services SPI and health sweeps. |
| PMU firmware acting alone | PMUFW suspends a PLL only when its request count drops to zero; the zero-count is Linux's doing (no DT consumer). |

## Fix (staged, not yet deployed)

`clk_ignore_unused` on the kernel command line — the kernel then skips the
unused-clock sweep entirely and RPLL stays locked. Placed in
`meta-monutchee/meta-zynqmp-addon/recipes-bsp/u-boot/u-boot-xlnx-scr.bbappend`:

```bitbake
KERNEL_COMMAND_APPEND:append:zynqmp = " clk_ignore_unused"
```

Scoping rationale: the mechanism requires the ZynqMP PS architecture
(firmware-owned PLLs + an RPU clockable from RPLL + no DT model of the RPU
clock), so it went in the family-wide `meta-zynqmp-addon` policy layer with
a `:zynqmp` machine override — not Kria-specific, not all-Xilinx (Zynq-7000
and pure-fabric parts lack the mechanism; Versal has an analogous but
distinct one). Cost on ZynqMP machines without RPU-from-RPLL designs: minor
idle power from clocks Linux would otherwise gate. The flag flows
`KERNEL_COMMAND_APPEND` → `boot.scr` → kernel cmdline, so rebuilding
`u-boot-xlnx-scr` and updating the boot partition carries it; the matched
boot-firmware redeploy (above) must ship with it to get 533 rather than 375.

Cleaner long-term alternatives if the blanket flag ever becomes
undesirable: source CPU_R5 from IOPLL in Vivado (~500 MHz, immune because
IOPLL always has consumers), or have the RPU firmware claim its own clock
through the XilPM API so PMU firmware refcounts it properly.

## How to verify after redeploy

```sh
cat /proc/cmdline                 # must contain clk_ignore_unused
devmem2 0xFF5E0030 w              # expect RESET/BYPASS clear, FBDIV=0x40
devmem2 0xFF5E0040 w              # expect bit1 (RPLL lock) = 1
devmem2 0xFF5E0090 w              # expect 0x0100_0200-style: RPLL/2 active
grep ' rpll ' /sys/kernel/debug/clk/clk_summary   # expect ~1066666650
```

A before/after of `RPLL_CTRL` early in boot (e.g. from an initramfs shell or
JTAG) versus post-boot would additionally confirm the suspend-at-boot
mechanism directly, if ever needed.

## Revisit if

- **RPLL reads suspended again after the fix** — something else is managing
  clocks (a kernel update changing `clk_ignore_unused` semantics, or a new
  PM feature); escalate to the XilPM-request or IOPLL-source fix.
- **The record-emission fault (2026-08-14 #2) changes character** after the
  R5 returns to full speed — in either direction, that is evidence about its
  root cause.
- **A Versal migration happens** — this flag does not carry the analysis
  over; the PLM-managed equivalent needs its own check.
