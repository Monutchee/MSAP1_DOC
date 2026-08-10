# MSAP1 PL timing closure: pipelining the Class A aggregator squaring path

## Document purpose

This document records the timing-closure defect found in the 150/180-cycle
Class A aggregator and the change that fixed it. The previous bitstream did
not meet setup timing: it shipped with a worst negative slack of −0.632 ns and
397 failing endpoints, every one of them inside `meter_cycle_aggregator`. The
fix pipelines the per-channel squaring step across three clocks so no single
path carries a 512-bit lane mux, a 64×64 multiply, and a 132-bit accumulate in
one AXI clock period.

The document covers:

- how the defect surfaced, and why the investigation that found it started
  somewhere else entirely;
- the failing path, stage by stage, from the routed timing report;
- the RTL construct that produced it, and why static analysis was needed to
  find it rather than simulation;
- the change, and the argument that the aggregated values are bit-identical;
- a hidden latency assumption in the testbench that the change exposed;
- before-and-after implementation results;
- **what this change does not fix**, which matters as much as what it does;
- traps for maintainers and the remaining timing headroom.

This is a corrective report against the aggregator described in:

```text
applications/MSAP1_DOC/raw_doc/impl_report/DOCPR00010_CLASS_A_150_180_CYCLE_AGGREGATION.md
applications/MSAP1_DOC/raw_doc/impl_report/DOCPR00009_CLASS_A_TIMING_FOUNDATION.md
```

The normative contracts remain:

```text
applications/MSAP1_PL/SourceData/DesignFile/MeterProcessing/README.md
applications/MSAP1_PL/SourceData/DesignFile/MeterCore/README.md
applications/MSAP1_APU/docs/TIMING_MODEL.md
```

If a bit-level detail differs between this report and those contracts or the
source, the contracts and the source are authoritative.

Change under review, 2026-08-10:

```text
MSAP1_PL   PR #20   commit 58c4582   branch Fix/sequence_gap_happen
                                     aggregator squaring pipeline + testbench gap
```

DOCPR00010 section 3.3 remains accurate on the arithmetic geometry and on the
finalize chain. Only the squaring walk length changes: 7 clocks per Basic
block becomes 21.

---

## 1. How the defect surfaced

The investigation did not start here. It started with a field symptom on the
production board that is **still open** and is not caused by this defect.

Between 2026-08-09 23:18 and 2026-08-10 08:34 the APU logged repeated
`meter_sample_range_gap` warnings. Two distinct episodes appeared, and the
second was dramatically worse than the first:

| Episode | Window | Corrupted records | `sequence_gaps` / `invalid_records` | Records | Rate |
| --- | --- | --- | --- | --- | --- |
| 1 | 08-09 23:18:32 → 23:19:14 (42 s) | 5 | 25 / 11 | 127,847 | 0.020 % |
| 2 | 08-10 08:26:50 → 08:34:08 (7 m 18 s) | 34 | 181 / 114 | 32,225 | 0.56 % |

The corrupted records were valid MTR1 v2 records in every field **except** the
64-bit first-sample index in words 60/61, which read zero, and — in five cases
in episode 2 — the nominal-frequency byte of the word-15 timing word, which
was rejected by the APU decoder as `invalid nominal frequency in MTR1 v2
timing word`. Nothing was lost from the sample stream: the record following
each corrupted one resumed at exactly `previous_first + previous_count`.

Two observations shaped what was investigated next.

**The corruption is locked to the aggregate boundary.** All 34 events in
episode 2 were separated by an exact integer multiple of 15 Basic blocks —
1×, 2×, 3×, 5×, 7×, 9×, 13×, and 16× — with no exception across 33 consecutive
intervals. Fifteen blocks is exactly one MTR2 aggregation period.

**The corrupted fields have a single common source.** Word 15 and words 60/61
are the only two fields in the MTR1 v2 record fed from `grid_cycle_timing`'s
`closed_*` register set, by way of the result hub's `block_first_sample_i`,
`block_cycle_count_i`, `block_nominal_hz_i`, and `block_flags_i` inputs. Every
other field in the record comes from the RMS engine, the frequency engine, or
the capture counters, and none of those ever corrupted.

That pointed at the aggregation boundary as the trigger and at the
`grid_cycle_timing` → `result_hub` handoff as the site. Because the aggregator
is the only module whose activity is periodic at the aggregate boundary, its
implementation was examined — and that examination found an unrelated but
entirely real defect: the bitstream did not close timing.

---

## 2. The failing path

`report_timing_summary` on the shipped routed design (built 2026-08-09
09:19:40, Vivado 2025.2, `xck26-sfvc784`, speed grade **−2LV**, temperature
grade C):

```text
WNS(ns)   TNS(ns)   TNS Failing Endpoints   TNS Total Endpoints
 -0.632  -123.823                     397                153724

Timing constraints are not met.
```

Hold was clean throughout (WHS +0.010 ns, zero failing endpoints); the defect
was setup only. The per-clock breakdown localised it completely:

| Clock | WNS | TNS | Failing endpoints | Total |
| --- | --- | --- | --- | --- |
| `ADC_DCLK` | +54.994 | 0.000 | 0 | 2,560 |
| `clk_pl_0` | **−0.632** | **−123.823** | **397** | 150,927 |

All 397 failures were on `clk_pl_0`, the 99.999 MHz AXI clock that carries the
whole metrology chain. Enumerating every failing endpoint from the routed
checkpoint — not just the ten paths the summary report prints — returned a
single structure:

```text
397 / 397 endpoints:  cycle_aggregator/square_acc_reg[*][*]
  0 / 397 endpoints:  arbiter, packetizer, result hub, or record data path
```

Worst slack −0.632 ns, best failing slack −0.007 ns. **The record bus closed
timing.** That result mattered: it eliminated an attractive but wrong theory
that the basic/aggregate handover on the shared 2048-bit combinational record
mux was being sampled mid-switch by the packetizer.

The worst path:

```text
Source:       cycle_aggregator/capture_rms_reg[133]/C
Destination:  cycle_aggregator/square_acc_reg[3][128]/D
Requirement:  10.000 ns      Data Path Delay: 10.634 ns
              logic 7.211 ns (67.8 %)   route 3.423 ns (32.2 %)
Logic Levels: 31  (CARRY8=12  DSP_MULTIPLIER=1  DSP_ALU=4  DSP_OUTPUT=4
                   DSP_A_B_DATA=1  DSP_M_DATA=1  DSP_PREADD_DATA=1
                   LUT6=5  LUT3=1  LUT2=1)
```

Walking the arrival times shows where the period went:

| Stage | Arrival | Cost | What it is |
| --- | --- | --- | --- |
| `capture_rms_reg[133]/Q` | 2.421 ns | — | source register, after 2.303 ns clock |
| LUT6 → LUT3 | 3.281 ns | 0.86 ns | 512 → 64 lane mux on `work_channel` |
| CARRY8 × 5 | 3.996 ns | 0.72 ns | two's-complement negation to `magnitude` |
| LUT6, route | 4.782 ns | 0.79 ns | into the DSP `A` port |
| DSP48E2 × 4 cascade | 10.634 ns | **5.85 ns** | 64×64 multiply and accumulate |

Over half the period was the DSP cascade. The `A_B_DATA → PREADD → MULTIPLIER
→ M_DATA → ALU → OUTPUT` chain crosses one DSP, then three further
`ALU → OUTPUT` stages ride the `PCOUT → PCIN` cascade at roughly 0.9 ns each.
A 64×64 → 128-bit product plus a 132-bit accumulate cannot fit in one
DSP48E2, so synthesis built a partial-product tree and cascaded it — and had
nowhere to pipeline it into, because the RTL offered no register boundary
between the multiply and the accumulator.

---

## 3. Why the design produced it

The original `S_SQUARE` state did four things in a single clock edge:

```vhdl
when S_SQUARE =>
  sample_q16 := signed(capture_rms(
    (work_channel * 64) + 63 downto work_channel * 64));   -- 512 -> 64 mux
  if sample_q16 < 0 then
    magnitude := unsigned(-sample_q16);                    -- negate
  else
    magnitude := unsigned(sample_q16);
  end if;
  square := magnitude * magnitude;                         -- 64 x 64 multiply
  square_acc(work_channel) <= square_acc(work_channel) +
    resize(square, AGGREGATE_ACCUMULATOR_BITS);            -- 132-bit accumulate
```

Both `magnitude` and `square` are VHDL **variables**, not signals. That is
correct and intentional for combinational intermediate values, and it reads
naturally — but it also means the entire mux → negate → multiply → accumulate
chain collapses into the logic cone of one registered assignment. The
structure was invisible in review precisely because it looked like four
ordinary sequential statements.

This class of defect is unreachable by simulation. The testbench asserted
values, and the values were always correct — behavioural simulation has no
notion of propagation delay. Only static timing analysis on the implemented
design could find it, and only enumerating the failing endpoints could prove
the record bus was innocent.

---

## 4. The change

The squaring step now spans three states, one register boundary each:

| State | Work | Path it owns |
| --- | --- | --- |
| `S_SQUARE_LOAD` | select the channel's Q16 lane, register its magnitude | 512 → 64 mux + negation |
| `S_SQUARE_MULT` | `square_product <= square_magnitude * square_magnitude` | the 64×64 multiply |
| `S_SQUARE_ACC` | add the product to the accumulator, advance the channel | 132-bit accumulate |

Two new signals replace the two removed variables:

```vhdl
signal square_magnitude : unsigned(63 downto 0)  := (others => '0');
signal square_product   : unsigned(127 downto 0) := (others => '0');
```

Neither is reset in the `aresetn` branch. That follows the existing convention
in this module — `divider_*` and `sqrt_*` are not reset either — and is safe
because both are written before they are read on every pass through the walk.

`S_SQUARE_ACC` keeps the original channel-advance and finalize logic
unchanged; the only addition is `state <= S_SQUARE_LOAD` on the
not-last-channel branch, since the walk is no longer a single self-looping
state.

**Cost.** The per-block walk grows from 7 clocks to 21. The relevant budget is
the Basic block interval:

```text
squaring walk    21 clocks   ~0.21 us   once per Basic block  (~200 ms)
finalize chain 1,961 clocks  ~19.6 us   once per aggregate    (~3 s)
```

Both remain four to five orders of magnitude inside their intervals. The
engine still has no ready output and still cannot backpressure ADC capture,
RMS, frequency, or MTR1 production. The `state /= S_IDLE` coherency guard that
catches a Basic result arriving mid-walk is unchanged and its margin is still
overwhelming — 0.21 µs of exposure in a 200 ms period.

---

## 5. Arithmetic invariance

The change is a pure retiming. Nothing about the computation moves:

- the lane selection is the same slice of the same `capture_rms` register;
- the magnitude is the same two's-complement negation of the same signed
  64-bit value;
- the product is the same full-width `unsigned(63:0) * unsigned(63:0)` →
  `unsigned(127:0)`, with `square_product` declared at exactly the natural
  result width so no truncation is introduced at the new register boundary;
- the accumulate is the same `resize(·, AGGREGATE_ACCUMULATOR_BITS)` addition
  into the same 132-bit accumulator, in the same channel order.

Because every stage of the arithmetic geometry in DOCPR00010 section 3.3 is a
floor operation on a width that cannot overflow by construction, and because
no width changed, the accumulated values are bit-identical to the single-cycle
form. The testbench confirms this empirically rather than by argument alone:
its wide-integer golden model asserts exact aggregate RMS values, and those
assertions pass unmodified.

---

## 6. A hidden latency assumption in the testbench

`meter_cycle_aggregator_tb` spaced its Basic result events like this:

```systemverilog
// Cover the 7-cycle square/accumulate walk before the next event.
repeat (12) @(posedge clock);
```

The comment names the coupling explicitly, which is why it was caught. At 21
clocks the next `basic_i.valid` would have landed while the engine was still
in the walk, hit the `state /= S_IDLE` branch, been counted as a coherency
loss, and discarded the partial aggregate. Every multi-block scenario in the
testbench would have failed — loudly, which is the good outcome, but it would
have looked like a functional regression in the change rather than a stale
constant in the bench.

The gap is now 26 clocks, five clocks of slack above the 21 the walk needs.

This is worth recording as a general hazard: a testbench that encodes an
internal implementation latency as a literal will break on any legitimate
retiming. The bench is the only consumer of that timing — `meter_core.vhd`
drives the aggregator through the record bus with full handshaking and is
indifferent to the walk length.

---

## 7. Verification

**Simulation.** `check_metering_pipeline.tcl` — all metering pipeline
simulations pass, including `meter_cycle_aggregator_tb`'s exact-value RMS
assertions against its golden model. Aggregated results are unchanged.

**Focused synthesis.** `check_metering_synthesis.tcl -tclargs
MeterCore_Wrapper` passes with 0 errors and 0 critical warnings. Out-of-context
timing reports WNS +2.088 ns with zero failing endpoints across 85,238
endpoints. Out-of-context numbers are optimistic — no real placement, no BD
context, ideal clock — so they were treated as a smoke test, not a result.

**Implementation.** The authoritative result, routed, 2026-08-10 09:22:

| Metric | Before | After |
| --- | --- | --- |
| WNS | −0.632 ns | **+0.295 ns** |
| TNS | −123.823 ns | **0.000 ns** |
| Setup failing endpoints | **397** | **0** |
| Total endpoints | 153,724 | 154,665 |
| WHS / THS | +0.010 / 0.000 | +0.011 / 0.000 |
| WPWS | +1.000 | +1.000 |
| `ADC_DCLK` WNS | +54.994 | +56.820 |
| Verdict | Timing constraints are not met | **All user specified timing constraints are met** |

`square_acc` has left the critical paths entirely. The new limiting path is:

```text
Slack (MET):  0.295 ns
Source:       rms_engine/snapshot_square_reg[3][3]/C
Destination:  rms_engine/variance_product_reg[127]/D
Path Group:   clk_pl_0
```

This was predicted before the change. Querying the pre-fix routed checkpoint
for the worst non-`square_acc` endpoint returned the same `variance_product`
path at +0.207 ns, so it was known to be next in line and known to be the
ceiling any aggregator fix would reach.

---

## 8. What this change does not fix

**The `meter_sample_range_gap` corruption from section 1 remains open.** The
branch name `Fix/sequence_gap_happen` reflects where the investigation began,
not what this commit resolves.

The reasoning is structural, not probabilistic. Words 60/61 and word 15 are
produced by the result hub from `grid_cycle_timing`'s `closed_*` registers.
The aggregator is a *consumer* of `basic_result`; it has no path to any field
of the MTR1 record, and `meter_core.vhd` confirms it drives nothing back into
grid timing. It was never a candidate for producing that corruption.

What closing timing does accomplish is removing a confound. A design with 397
failing endpoints on the metrology clock is not a valid basis for diagnosing
an intermittent fault anywhere in that clock domain, because any observation
could be an artifact of the violation. The theory under which the two are
linked — that the aggregate boundary is the peak-switching instant for
`cycle_aggregator`, and that dynamic margin degradation there could disturb a
neighbouring handoff — is plausible, was never proven, and is now testable by
elimination.

Candidates ruled out during the investigation, recorded so they are not
re-litigated:

| Hypothesis | Why it was eliminated |
| --- | --- |
| Aggregate record misclassified as MTR1 v2 | Word 6 read 25,606 — a Basic block count, not the ~384 k aggregate total |
| PL/APU word-offset ABI mismatch | `MTR1_*`/`MTR2_*` constants match the APU accessors exactly |
| Aggregate producer emitting the wrong format ID | Writes `MTR2_FORMAT` correctly and zero-fills the rest |
| Record-bus mux sampled mid-switch by the packetizer | The packetizer latches all 2048 bits in one clock; 0 of 397 failing endpoints were on that path |
| `grid_cycle_timing` reset or APPLY | `closed_nominal_hz` resets to 60, not 0; APPLY does not touch `closed_*` |
| TUSER high-word slice `user(105 downto 74)` | Matches the documented layout in `metering_pkg.vhd` |
| DMA, DDR bandwidth, or cache coherency | The waveform channel moved 22.4 GB over the same window with zero gaps, invalid blocks, or overruns |

**Next step if the corruption persists.** Watch `sequence_gaps` in
`mnc --output json meter health` over several hours on the new bitstream —
episode 1 took over six hours to appear, so a short quiet window proves
nothing. If gaps still accumulate at exact 3-second multiples, the fault is
functional RTL in the `grid_cycle_timing` → `result_hub` provenance handoff,
and the decisive instrument is an ILA on `closed_first_sample` and
`block_nominal_hz` sampled against `result_valid` across an aggregate
boundary.

---

## 9. Traps for maintainers

**Wide arithmetic in a single registered assignment is invisible in review and
in simulation.** The original `S_SQUARE` looked like four ordinary sequential
statements. Any future state that combines a wide mux, a multiply, and an
accumulate into one clock will reproduce this defect, and no testbench will
report it. The same pattern exists today in `rms_engine`
(`variance_product`) and in this module's `S_SQRT_MULTIPLY`
(`sqrt_square <= midpoint * midpoint`, fed by a 65-bit add on `sqrt_low` and
`sqrt_high`). Both currently close timing; neither has much margin.

**Implementation is not optional for metrology RTL changes.** Simulation and
focused synthesis both passed on the defective design. Only
`implement_ad7771_design.tcl` and a review of the routed timing summary would
have caught it, which is exactly what the PL `AGENTS.md` verification ladder
requires before handing a new XSA to RPU/Yocto. Treat a non-zero
`TNS Failing Endpoints` as a blocking result, not a warning.

**The summary report prints ten paths, not all of them.** With
`-max_paths 10`, `report_timing_summary` showed only the ten worst violations,
all in one block. Concluding anything about which structures are *not* failing
requires enumerating endpoints from the routed checkpoint —
`get_timing_paths -setup -slack_lesser_than 0` — because 387 of the 397
failures were invisible in the report. An early misreading of that report
attributed the worst paths to the `ADC_DCLK` group, which the intra-clock
table contradicts.

**`square_magnitude` and `square_product` are not reset.** This matches the
module's existing convention and is safe only because the walk always writes
before reading. A future change that reads either signal outside the
`S_SQUARE_*` sequence must establish its own initialisation.

**The testbench inter-event gap tracks the walk length.** If the walk changes
length again, `repeat (26)` in `meter_cycle_aggregator_tb` must grow with it,
or every multi-block scenario will fail through the coherency-reset path.

---

## 10. Remaining headroom and future work

**+0.295 ns is closed, not comfortable.** On a −2LV part at 99.999 MHz that is
roughly 3 % of the period, and it is dominated by a single structure. Deferred
deliberately, so that the hardware test isolated one variable:

- **Pipeline `rms_engine`'s `variance_product`.** Same pattern — a wide
  multiply feeding a register in one cycle — and the same three-stage split
  applies. This is the only change that raises WNS materially today.
- **Revisit `S_SQRT_MULTIPLY`.** A 65-bit add feeding a 64×64 multiply into
  `sqrt_square` in one clock. It closes now, but it is the same shape as the
  defect this report describes, and the sqrt loop runs 64 times per channel so
  a retiming there costs 64 extra clocks per channel — still negligible
  against a 3-second interval.
- **Consider a timing-closure gate in the verification ladder.** A scripted
  assertion that `TNS Failing Endpoints` is zero after implementation would
  have prevented a non-closing bitstream from reaching hardware at all.

**Explicitly not planned.** Moving the aggregator to HLS was considered and
rejected. HLS would have prevented this specific defect, since it schedules to
a target period, but roughly two thirds of this module is eligibility,
continuity, and diagnostic state coupled to the `measurement_record_bus_pkg`
wire contract rather than arithmetic — the part HLS expresses worst. It would
also degrade exactly the debugging the open corruption needs, since probing
machine-named signals in a scheduled datapath is materially harder than
probing `closed_first_sample` in hand-written VHDL. If an HLS boundary is ever
wanted here, the clean seam is the finalize math alone — seven 132-bit
accumulators and `freq_sum` in, seven 64-bit RMS values and the frequency mean
out, one invocation per three seconds. A future harmonics or flicker engine,
attached to the record bus as a new arbiter producer, is a far better first
HLS candidate.
