# MSAP1 M20 IEC 61000-4-30 UTC 10-second frequency implementation report

| Field | Value |
| --- | --- |
| Document ID | DOCPR00021 |
| Milestone | M20 UTC-aligned 10-second frequency |
| Report date | 2026-09-02 |
| Status | Post-implementation report with PL, RPU, APU, Web, Yocto, block-design, and target acceptance evidence |
| Product baseline | 128 kSPS ADC capture; 50 Hz or 60 Hz nominal configuration; certified M20 profile uses voltage channel CH6, filter profile 1, and calibration profile 1 |
| Scope | IEC 61000-4-30 Class A ten-second frequency, UTC/sample correlation, dedicated meter-time control, R5C1 result authority, persistence, API, operator/developer UI, NTP/PTP selection, packaging, and verification |

## 1. Purpose

M20 adds an independent, UTC-aligned, ten-second grid-frequency product. It
replaces the earlier informative frequency values that happened to accompany
Basic or 150/180-cycle records with a result whose interval, clock quality,
frequency geometry, calculation, and rejection reasons can be audited as one
coherent measurement.

The implementation has four primary goals:

- measure frequency over exact UTC ten-second intervals `[start, end)`;
- keep signal observation deterministic and nonblocking in programmable logic;
- keep one calculation and public-record authority in R5 core 1; and
- expose valid and rejected intervals without inventing a numeric result when
  the Class A timing or signal-quality requirements are not met.

M20 also removes metrology time control from the waveform DMA register bank.
The new `/dev/meter-time` path is a small register-only control endpoint. It
does not allocate a DMA channel and does not transfer sample payloads.

This report describes the implemented branch state and the final target check.
Source code and the PL/R5 contract headers remain authoritative for bit-level
details.

## 2. Delivered outcome

| Capability | Implementation | Result |
| --- | --- | --- |
| Standard interval | UTC boundaries whose nanoseconds are divisible by 10 seconds | Every result names one exact half-open UTC interval |
| Signal source | Certified voltage reference CH6 (VLA) | Frequency is independent of the Basic and 150/180-cycle frequency fields |
| PL observation | Fixed conditioner, positive-going crossing interpolation, guarded interval capture | Linux scheduling cannot move a detected crossing |
| Result authority | R5C1 `Frequency10sEngine` | One implementation selects complete cycles, calculates frequency, and creates the record |
| Public record | Fixed 256-byte `FREQUENCY-10S-v1`, format `0x00280001` | Valid and invalid intervals share one ordered, auditable sequence |
| UTC control | Dedicated `meter-time-control` AXI4-Lite block and `/dev/meter-time` driver | Waveform DMA no longer owns clock or interval-control operations |
| Time qualification | Kernel-bracketed PL correlation plus PL elasticity and Linux clock-discipline estimates | A synchronized clock is distinguished from a Class A-qualified interval |
| Persistence | Durable stream and persistent `seconds_10` historian dataset | Completed results survive service restart and are queryable by period |
| API | `GET /api/v1/meter/frequency-10s` | Exact value, UTC interval, uncertainty, geometry, profile, status, and reasons are available |
| Operator UI | Data tab, measurement interval `10 seconds` | Operators see a compact frequency and validity presentation |
| Developer UI | Developer → Time | Detailed clock, sample, crossing, profile, sequence, and rejection audit remains available |
| Clock selection | NTP default; optional PTP on `end0` | Existing installations keep NTP while suitable networks can select hardware-timestamped PTP |

## 3. Design concept

### 3.1 One measurement, two planes

M20 separates the implementation into a control plane and a measurement-data
plane.

The control plane maps Linux UTC into the PL sample domain:

```text
NTP or PTP -> CLOCK_REALTIME/adjtimex -> kernel-bracketed PL latch
           -> APU boundary mapping -> AXI4-Lite meter-time registers
```

The measurement-data plane observes the voltage, calculates the result, and
publishes it:

```text
CH6 samples -> PL conditioner/crossing observer -> FRQ1 observation
            -> R5C1 calculation -> FREQUENCY-10S-v1
            -> existing meter DMA -> APU stream/history/API -> Web
```

The split is intentional. UTC discipline and policy belong to Linux, while
sample-accurate observation cannot depend on Linux task scheduling. The R5C1
firmware remains the common interval-result authority introduced by the
aggregation offload architecture.

### 3.2 Why the complete calculation is not on the APU

Calculating the measurement from Linux callbacks would introduce avoidable
scheduler, device-I/O, and service-latency dependence into interval closure.
The APU also does not receive the raw CH6 sample stream on the meter-record
path. Reconstructing the calculation there would create a second metrology
implementation and weaken continuity evidence.

Instead:

- PL observes every accepted conversion frame and never waits for software;
- PL preserves interpolated crossing positions and the exact boundary
  provenance;
- R5C1 validates the complete observation and performs deterministic integer
  arithmetic; and
- APU validates, stores, and presents the resulting fixed public record
  without recomputing or retimestamping it.

### 3.3 Why the new block is not a DMA

The APU sends only a small boundary tuple approximately once per ten seconds.
AXI4-Lite registers are the natural transport for that transaction. A DMA
would add descriptors, interrupts, buffers, lifecycle state, and another data
owner without increasing correctness or throughput.

The existing meter-record DMA is receive-only measurement transport owned by
Linux. It remains the route for completed 256-byte results. The private
PL↔R5C1 AXI FIFO remains the route for `FRQ1` observations and returned meter
records. Neither interface is repurposed for APU-to-PL time configuration.

## 4. Release provenance and integration set

The implementation is published from `feat/m20_freq_10s` through five pull
requests:

| Repository | Pull request | Inspected branch tip | M20 responsibility |
| --- | --- | --- | --- |
| `MSAP1_PL` | #39 | `cbac6ea` | Conditioner, crossing observer, `FRQ1` transport, dedicated time-control registers, and `S_AXI_TIME` block-design connection |
| `MSAP1_RPU` | #28 | `5d5a169` | `FRQ1` validation, ten-second calculation, fixed record construction, sequence continuity, and health |
| `MSAP1_APU` | #59 | `5e4ae2d` | Time correlation/boundary scheduling, decoder, durable publication, historian, API, settings, and NTP/PTP control |
| `MSAP1_WEB` | #32 | `5c8e8be` | Operator ten-second interval, Developer time audit, and synchronization selector |
| `meta-monutchee` | #61 | `edb53eb` | Device tree, `meter_time.ko`, udev, NTP tuning, linuxptp, service integration, and image packaging |

The repositories are co-released. A mixed PL/R5 image is rejected by the
private frame contract and CRC checks; a Linux image without the `MTC1`
register block does not bind `/dev/meter-time`.

## 5. System architecture

### 5.1 Component architecture

~~~mermaid
flowchart LR
    subgraph CLOCK["Linux clock and control plane"]
        NTP["systemd-timesyncd<br/>NTP default"]
        PTP["ptp4l + phc2sys<br/>optional PTP on end0"]
        KCLK["CLOCK_REALTIME / CLOCK_TAI<br/>adjtimex discipline state"]
        DRIVER["meter_time.ko<br/>/dev/meter-time"]
        COORD["CaptureCoordinator<br/>correlation, uncertainty, boundary mapping"]
        NTP --> KCLK
        PTP --> KCLK
        KCLK --> COORD
        COORD --> DRIVER
    end

    subgraph PL["Programmable logic"]
        TIME["meter-time-control<br/>MTC1 AXI4-Lite registers"]
        ADC["AD7771 accepted frames<br/>sample index + CH6 voltage"]
        CONDITION["CIC + linear-phase FIR<br/>profile 1 conditioner"]
        OBSERVE["10-second observer<br/>hysteresis + Q16 crossings"]
        PACK["FRQ1 packetizer<br/>whole-packet reservation + CRC32C"]
        ADC --> CONDITION --> OBSERVE --> PACK
        TIME --> OBSERVE
    end

    subgraph RPU["R5 core 1"]
        FIFO["Shared aggregation AXI FIFO"]
        VALIDATE["Frequency10sFrameDecoder"]
        ENGINE["Frequency10sEngine<br/>complete-cycle selection + arithmetic"]
        RECORD["FREQUENCY-10S-v1<br/>256-byte record"]
        FIFO --> VALIDATE --> ENGINE --> RECORD
    end

    subgraph LINUX["Linux data plane"]
        DMA["Existing meter-record DMA"]
        ACQ["msap1-fpga-acquisition<br/>strict decode + durable publish"]
        STREAM["msap1-meter-stream"]
        HIST["msap1-meter-historian<br/>seconds_10"]
        API["Web backend<br/>/api/v1/meter/frequency-10s"]
        DMA --> ACQ --> STREAM --> HIST
        ACQ --> API
        HIST --> API
    end

    subgraph UI["Presentation"]
        DATA["Data → 10 seconds<br/>operator view"]
        DEV["Developer → Time<br/>full audit"]
    end

    DRIVER -->|"M10_AXI at 0xB00A0000"| TIME
    PACK --> FIFO
    RECORD -->|"R5 FIFO return"| DMA
    API --> DATA
    API --> DEV
~~~

### 5.2 Ownership boundaries

| Concern | Owner | Constraint |
| --- | --- | --- |
| ADC capture and monotonic sample index | PL | Never stepped by UTC and never backpressured by the ten-second observer |
| UTC discipline | Linux NTP/PTP services and kernel | Clock selection is settings policy; NTP remains the factory default |
| PL↔UTC correlation and boundary programming | Acquisition daemon through `/dev/meter-time` | One exclusive userspace owner; register operations only |
| Signal conditioning and crossing timestamps | PL | Deterministic, fixed profile, and conversion-sample coordinates |
| Complete-cycle selection and frequency arithmetic | R5C1 | Sole production result authority |
| Fixed meter-record DMA and DDR buffers | Linux kernel/APU | Existing receive path; no new DMA for M20 control |
| Record validation, durability, and latest state | APU | UTC endpoints in the record are immutable and are not restamped |
| Presentation | Web backend and Web UI | Operator summary is separate from developer audit detail |

R5 core 0 retains ADC SPI/configuration ownership. RPMsg remains control and
health only; no M20 observation or result payload travels over RPMsg.

## 6. End-to-end system flowchart

~~~mermaid
flowchart TD
    START["Acquisition starts or reaches a 10 s refresh"] --> SAMPLE["Issue three correlation ioctls"]
    SAMPLE --> LATCH["Kernel brackets an atomic PL tick/sample latch<br/>with CLOCK_TAI and CLOCK_REALTIME"]
    LATCH --> BEST["Select the correlation with the narrowest bracket"]
    BEST --> DISCIPLINE{"Kernel clock synchronized?"}
    DISCIPLINE -->|"No"| MAPBAD["Create next boundary with<br/>time_synchronized = false"]
    DISCIPLINE -->|"Yes"| UNCERTAINTY["Combine bracket, 16-frame elasticity,<br/>PLL offset, esterror, and precision"]
    UNCERTAINTY --> MAPGOOD["Map the next UTC multiple of 10 s<br/>to [start_sample, end_sample)"]
    MAPBAD --> COMMIT["Commit coherent tuple through /dev/meter-time"]
    MAPGOOD --> COMMIT
    COMMIT --> QUEUE["PL queues/activates boundary tuple"]
    QUEUE --> CONDITION["Condition CH6 and detect guarded<br/>positive-going Q16 crossings"]
    CONDITION --> CLOSE["At end_sample freeze observation and provenance"]
    CLOSE --> FRQ1["Packetize FRQ1 with CRC32C"]
    FRQ1 --> DECODE{"R5C1 validates frame and continuity"}
    DECODE -->|"Malformed or unrecoverable"| HEALTH["Raise aggregation health fault"]
    DECODE -->|"Accepted"| CALC["Select in-interval crossings,<br/>check every cycle, calculate frequency"]
    CALC --> QUALIFY{"All profile, time, signal,<br/>geometry, and transport checks pass?"}
    QUALIFY -->|"Yes"| VALID["Emit numeric valid result"]
    QUALIFY -->|"No"| INVALID["Emit zero-valued rejected interval<br/>with exact reason bits"]
    VALID --> RECORD["FREQUENCY-10S-v1"]
    INVALID --> RECORD
    RECORD --> DMA["Existing meter-record DMA"]
    DMA --> VERIFY["APU cross-validates arithmetic,<br/>status, geometry, and UTC"]
    VERIFY --> DURABLE["Durable stream before latest publication"]
    DURABLE --> HISTORY["Persistent seconds_10 projection"]
    DURABLE --> API["Frequency 10 s API"]
    HISTORY --> API
    API --> OPERATOR["Operator Data view"]
    API --> DEVELOPER["Developer Time audit"]
~~~

## 7. UTC correlation and boundary control

### 7.1 Dedicated meter-time endpoint

`meter_time_control_axi_regs.vhd` is a standalone AXI4-Lite slave with
identifier `MTC1` (`0x3143544D`). It observes `sample_accept_i` and the
conversion sample index directly. It also owns a free-running 64-bit PL tick.
Neither value depends on meter DMA or waveform DMA progress.

The Linux platform driver:

- binds to `compatible = "monutchee,meter-time-control"`;
- verifies the `MTC1` identifier before registering;
- exposes `/dev/meter-time` as an exclusive-open misc device;
- uses mode `0660` and the `mnc-data` group;
- serializes operations with a mutex; and
- provides correlation, ten-minute boundary, ten-second boundary, and cancel
  ioctls.

The new module is named `meter_time.ko`; no product prefix is added. The
endpoint is deliberately neutral because the register contract is a metering
function, not a DMA implementation detail.

### 7.2 Atomic correlation

For one correlation ioctl, the driver performs this sequence in kernel
context:

1. read CLOCK_TAI and CLOCK_REALTIME before the PL access;
2. toggle the PL latch command;
3. read the atomic latched PL tick and sample index;
4. read CLOCK_REALTIME and CLOCK_TAI after the PL access; and
5. return both brackets and the PL values to userspace.

The acquisition daemon takes three correlations and selects the narrowest
REALTIME bracket. The midpoint supplies the UTC anchor. Taking several bounded
samples reduces bus/scheduler outliers without claiming that the bracket is
zero.

### 7.3 Clock uncertainty model

For M20, the reported conservative uncertainty is:

```text
Utotal = Uioctl-bracket
       + UPL-elasticity
       + |kernel PLL phase offset|
       + kernel estimated error
       + kernel clock precision
```

The PL correlation latch precedes a 16-frame elasticity FIFO, so:

```text
UPL-elasticity = 16 frames / active sample rate
```

At 128 kSPS that contribution alone is 125,000 ns. A stable displayed value
near 129 µs is therefore expected: approximately 125 µs of deliberately
conservative PL elasticity plus the much smaller ioctl bracket and disciplined
clock estimate. It is not evidence of a 129 µs measured clock offset.

The kernel `STA_UNSYNC` flag is a separate hard gate. A small numeric estimate
cannot qualify a clock that Linux reports as free-running.

### 7.4 Boundary mapping

The daemon refreshes correlation at a ten-second cadence and maps the next UTC
multiple of ten seconds into the sample domain. It uses the measured rate
between consecutive correlations when available. The tuple contains:

- exclusive start and end sample indices;
- exact UTC start and end nanoseconds;
- combined UTC uncertainty;
- measured sample rate in millihertz;
- a nonzero boundary generation;
- nominal frequency, reference channel, filter, and calibration profiles; and
- boundary-valid and clock-synchronized flags.

The first post-start tuple may use the configured sample rate only to preserve
sound geometry. It is deliberately marked rate-invalid until two correlations
provide a measured rate, so warm-up cannot become a valid result.

The PL register bank uses shadow fields and an update toggle. Linux writes the
whole tuple, commits once, reads it back, and checks that the toggle changed.
Capture stop or reconfiguration issues a cancel command so a stale boundary
cannot cross into the next measurement epoch.

### 7.5 Register groups

| Offset range | Purpose |
| --- | --- |
| `0x00`–`0x04` | Version and `MTC1` identifier |
| `0x08`–`0x1C` | Latch command/status and atomic latched PL tick/sample index |
| `0x20`–`0x2C` | Live diagnostic PL tick/sample index |
| `0x40`–`0x48` | Existing UTC ten-minute target and commit toggle |
| `0x50`–`0x84` | Ten-second sample/UTC endpoints, uncertainty, measured rate, generation, profile, and control |
| `0x88`–`0x98` | Observer status plus completed, dropped, overflow, and discontinuity counters |

The register block preserves ten-minute boundary control while removing all
time functions from the waveform register bank.

## 8. PL signal observation

### 8.1 Certified conditioner

The M20 observer consumes accepted converted frames and extracts CH6 voltage
in calibrated microvolts. The fixed profile is:

- third-order CIC decimation by 128, from 128 kSPS to 1 kSPS;
- symmetric 21-tap binomial FIR `(1 + z^-1)^20 / 2^20`;
- exact combined group-delay compensation back into conversion-sample
  coordinates; and
- 24 filtered-output warm-up before `filter_ready` can be asserted.

The filter state resets on configuration generation/rate discontinuity. A
reset is surfaced as a discontinuity; it is never silently bridged.

### 8.2 Crossing observer

The observer applies the certified hysteresis profile and detects
positive-going crossings. Each crossing is linearly interpolated and stored as
a signed Q16 sample offset relative to the interval start. Signed offsets allow
one guard crossing on either side of the boundary without changing the
two-word crossing representation.

Important implementation properties are:

- a capacity of 1,024 crossings per observation;
- double-buffered block RAM for capture and transport;
- exact start/end and guard-crossing flags;
- fixed metadata and zero padding;
- explicit filter, reference, sample-rate, profile, discontinuity, overflow,
  and observer-drop state; and
- no backpressure from the observer into ADC conversion.

When output storage is congested, the packetizer reserves capacity for a
whole observation. It may discard a complete `FRQ1` packet and increment an
observable counter, but it cannot emit a partial packet or stall metrology.

### 8.3 FRQ1 private transport

`FRQ1` is a PL-to-R5C1 observation contract, not the public meter record.

| Field | Value |
| --- | --- |
| Magic | `0x31515246` (`FRQ1`) |
| Contract revision | 1 |
| Total words | 2,077 |
| Metadata | 24 words |
| Crossing capacity | 1,024 signed Q16 positions, two words each |
| Integrity | CRC32C over the header and payload |

The packet shares the existing R5 aggregation AXI FIFO with `AGG1`, `PQE1`,
`VSB1`, and `HRM1`. A five-input arbiter switches only at `TLAST`, so packet
families cannot interleave.

## 9. PL block design and isolation

The final Vivado connection is:

```text
ZYNQ_System/M10_AXI
    -> MeterLogic/S_AXI_TIME
    -> Data_Computation/S_AXI_TIME
    -> MeterCore_Wrapper/S_AXI_TIME
```

The address editor maps the 64 KiB register window at `0xB00A0000`. The
SmartConnect exposes eleven masters (`M00_AXI` through `M10_AXI`): the new
meter-time endpoint uses M10, while M7 remains occupied by the waveform DMA
control path. This prevents accidental aliasing between waveform transport and
time control.

Vivado block-design validation succeeded after the connection was added. The
generated design shows the interface, clock/reset association, route, and
address segment consistently through the hierarchy.

## 10. R5C1 interval authority

### 10.1 Frame validation

`Frequency10sFrameDecoder` validates before exposing an immutable view:

- magic, contract revision, fixed extent, and mirrored sequence;
- CRC32C and reserved/zero padding;
- profile and metadata bit masks;
- interval and UTC geometry;
- crossing count and strictly ordered Q16 positions; and
- guard-crossing representation.

A malformed private frame cannot be converted into a plausible public record.
Validation and calculation run in task context; the interrupt path remains
bounded.

### 10.2 Frequency arithmetic

R5C1 selects crossings that lie inside the exact sample interval. For every
adjacent included pair it verifies a plausible cycle duration for the nominal
profile. Any rejected cycle invalidates the interval.

For `N` complete cycles spanning `Dq16` Q16 samples and a measured sample rate
`Fs_mHz`, the public frequency is:

```text
frequency_mHz = roundTiesToEven((Fs_mHz << 16) * N / Dq16)
```

The implementation checks every multiplication/division for overflow. The
accepted result range is 42.5–57.5 Hz for a 50 Hz nominal profile and
51–69 Hz for a 60 Hz nominal profile.

This is a complete-cycle, cycle-weighted result. It is not an average of the
Basic-block or 150/180-cycle frequency fields.

### 10.3 Sequence continuity

M20 has its own result sequence. It does not share continuity state with
Basic, aggregate, energy, harmonic, Flicker, or mains-signalling records.

For bounded forward gaps, R5C1 emits explicit zero-valued transport-gap
placeholders so the UTC sequence remains auditable. More than 64 missing
observations or an unrecoverable output failure makes the engine unhealthy
rather than hiding an unbounded loss.

## 11. FREQUENCY-10S-v1 public record

The public format is `0x00280001`, exactly 256 bytes. It retains the common
meter-record header and includes:

- independent result sequence and configuration generation;
- configured and measured sample rates;
- half-open sample interval and inclusive last-sample audit value;
- exact UTC start/end and uncertainty;
- frequency in millihertz, complete-cycle count, and Q16 duration;
- first/last in-interval crossing positions;
- source observation sequence and boundary generation;
- source status, result status, and rejection-reason masks;
- observed/included crossings and rejected-cycle count;
- observer drop count and guard flags; and
- nominal frequency, reference channel, filter, and calibration profiles.

Only a record with no rejection reason sets `result_valid`, CH6 in the valid
mask, and a numeric frequency. A rejected interval carries zero in the numeric
field while retaining its interval and diagnostic evidence.

## 12. APU implementation

### 12.1 Acquisition ownership

`msap1-fpga-acquisition` is the sole `/dev/meter-time` owner. It keeps the
descriptor open across the capture lifetime, correlates every ten seconds,
programs the next boundary, and cancels it during stop/reconfiguration.

Waveform metadata now obtains its correlation through the same independent
owner. The waveform DMA driver no longer implements time-control ioctls; old
operations return `ENOTTY` and current software uses the neutral meter-time
UAPI.

### 12.2 Strict decode and immutable measurement time

The APU decoder does not trust the public record merely because its header is
valid. It independently checks:

- UTC span is exactly ten seconds;
- sample, crossing, and cycle counts agree;
- Q16 duration equals last crossing minus first crossing;
- profile, status, and reason masks are legal;
- reason bits agree with the represented faults;
- frequency recomputation matches the R5C1 value; and
- valid mask, numeric value, status, and quality are mutually consistent.

The UTC endpoints and uncertainty are part of the M20 measurement record.
Unlike other records whose UTC is attached at ingest, M20 is never restamped
from the daemon's current timebase.

### 12.3 Durable publication and history

An accepted result is committed to `msap1-meter-stream` before latest-state
publication. The historian projects the `Seconds10` period into the persistent
`seconds_10` dataset. The default policy has no age or byte limit, matching the
existing long-interval compliance datasets.

The scalar frequency follows the canonical meter-attribute path. The companion
`Frequency10sMetadata` retains the exact interval, clock, observer, crossing,
profile, status, and reason audit.

### 12.4 REST contract

`GET /api/v1/meter/frequency-10s` returns either an unavailable response or one
complete result. The available response includes:

- `valid` and numeric `frequency_hz` only when publishable;
- UTC and sample interval endpoints;
- clock synchronization, Class A qualification, and uncertainty;
- configured/measured sample rates;
- cycle and crossing geometry;
- source/result sequences and boundary generation;
- profile identity;
- named source-status flags, result-status flags, and rejection reasons; and
- age, observer drops, and rejected cycles.

Invalid intervals are normal measurement outcomes and return their audit data;
they are not converted into transport errors.

## 13. Operator and developer Web UI

The primary live page is named **Data**. Its measurement-interval selector now
contains **10 seconds**. The operator view intentionally shows only the
information needed to use the measurement:

- current grid frequency;
- valid/rejected state;
- UTC-aligned interval context;
- clock qualification; and
- a short reason when no numeric value can be published.

The large diagnostic banner and exact calculation details live under
**Developer → Time**. That panel exposes the UTC/sample intervals, uncertainty,
measured rate, cycle/crossing geometry, profiles, generations, sequences,
status masks, and rejection reasons.

This split avoids presenting low-level metrology provenance as the main
operator workflow while keeping every diagnostic available to commissioning
and development users.

## 14. Validity and failure containment

A numeric result is published only if all checks pass. The following table
summarizes the principal rejection behavior.

| Condition | Result behavior |
| --- | --- |
| Linux clock reports unsynchronized | Record retained; numeric value suppressed; `time_unsynchronized` |
| Combined UTC uncertainty exceeds 1,000,000 ns | Record retained; numeric value suppressed; `time_uncertainty` |
| First rate measurement unavailable | Warm-up record retained; numeric value suppressed; `sample_rate_invalid` |
| Unsupported rate/channel/filter/calibration | Record retained; numeric value suppressed; profile/calibration reason |
| Filter not ready or CH6 invalid | Record retained; numeric value suppressed |
| Capture/configuration discontinuity | Record retained; numeric value suppressed; discontinuity status |
| Crossing buffer overflow or whole observation drop | Record retained when possible; numeric value suppressed; counters identify the source |
| Too few crossings | Record retained; numeric value suppressed; `insufficient_crossings` |
| Implausible individual cycle | Record retained; numeric value suppressed; `cycle_geometry` |
| Frequency outside the nominal profile range | Record retained; numeric value suppressed; `out_of_range` |
| FRQ1 CRC/layout failure | Private frame rejected and health/continuity updated; never decoded as a result |
| Public-record arithmetic/status mismatch | APU rejects the record before durable/latest publication |

The core principle is visible failure. A bad interval is represented as a bad
interval, while corrupt framing is quarantined. Neither becomes a plausible
numeric measurement.

## 15. NTP and PTP synchronization

### 15.1 NTP remains the default

Settings schema 7 introduces:

```json
"time": {
  "synchronization": "ntp"
}
```

The allowed values are `ntp` and `ptp`. Existing settings migrate to NTP, and
the factory document selects NTP. The image tunes `systemd-timesyncd` with a
32-second minimum poll, 256-second maximum poll, and 60-second save interval.

### 15.2 Optional hardware-timestamped PTP

Selecting PTP stops `systemd-timesyncd` and starts:

```text
ptp4l@end0.service
phc2sys@end0.service
```

The packaged `ptp4l` profile is client-only, UDPv4, E2E delay, domain 0, with
hardware timestamps. Selecting NTP performs the inverse transition. The
service manager periodically reconciles the active settings, so the selected
policy is restored after a service restart.

PTP can reduce phase-error excursions on a suitable IEEE 1588 network, but it
is not automatically superior when no grandmaster or correct network support
exists. That is why it is optional rather than replacing NTP.

### 15.3 Why uncertainty can move while the clock stays synchronized

`synchronized` means the kernel clock is disciplined. `Class A qualified`
additionally means the conservative per-interval uncertainty is no greater
than 1 ms. NTP can apply a normal phase correction while remaining
synchronized; the `adjtimex` offset term then raises the bound temporarily.

During target observation, uncertainty moved from about 852 µs through
747, 656, 580, 513, 457, 410, and 369 µs before returning near 143 µs. The
clock remained synchronized and the observed intervals remained below the
1 ms Class A limit. If the same bound crosses 1 ms, that interval is correctly
rejected and later intervals automatically resume when the estimate recovers.

## 16. Relationship to 10-minute and 2-hour aggregation

The ten-minute and two-hour products and M20 do not use identical timing
qualification semantics.

Long-interval aggregation is driven by an already-seeded sample-domain UTC
target and then advances deterministically with aggregation state. Its UI time
quality can remain synchronized across a short clock-estimate excursion.

M20 programs and audits each short interval with an explicit UTC uncertainty
and applies a hard 1 ms gate to that interval. Therefore it is possible, and
expected, for ten-minute/two-hour time status to look good while one ten-second
frequency interval is rejected. That difference is stricter per-interval
evidence, not a contradiction between clocks.

## 17. Packaging and deployment

The Yocto integration adds:

- `meter-time_1.0.bb`, building `meter_time.ko` and its UAPI;
- a device-tree node at `0xB00A0000`, size 64 KiB;
- a udev rule assigning `/dev/meter-time` to `mnc-data`, mode `0660`;
- `meter-time-sync`, installing the timesyncd tuning;
- linuxptp and the hardware-timestamped `ptp4l.conf`;
- the time-sync service definitions consumed by the service manager; and
- all packages in the MSAP1 image.

The waveform kernel module is reduced to waveform transport. Its legacy time
definitions remain only as compatibility documentation and reject current
operations.

## 18. Verification

### 18.1 PL and block design

- `meter_frequency_10s_transport_tb`: pass.
- `meter_time_control_regs_tb`: pass.
- Metering module reference check: pass.
- Focused MeterCore elaboration/simulation and spectral checks: pass.
- Focused synthesis: pass with zero errors and zero critical warnings.
- Vivado block-design validation after the M10 connection: pass.
- Address, clock/reset, and hierarchical `S_AXI_TIME` connection inspection:
  consistent.

The PL tests cover guarded boundary crossings, exact endpoints, continuous
handoff, cancel behavior, filter warm-up, backpressure, whole-packet drops,
CRC framing, and observer diagnostics.

### 18.2 RPU and APU

- R5C1 aggregation-shadow tests include `FRQ1` decoding, calculation,
  validity, placeholder, and continuity cases: pass.
- APU host build: pass.
- APU CTest suite: 49/49 pass.
- Decoder tests independently recompute the frequency and reject inconsistent
  status, reason, UTC, profile, and geometry fields.
- Acquisition tests verify independent sequence tracking, durable-before-latest
  ordering, and `seconds_10` routing.
- UTC clock tests cover synchronized state, nanosecond/microsecond offset
  conversion, saturation, and the uncertainty composition.

### 18.3 Web and Yocto

- Web TypeScript project build (`tsc -b`): pass.
- Operator/developer component tests were added for valid, unqualified,
  rejected, and unavailable states.
- Full local Vitest/Vite execution was blocked by the host's Node 18 runtime;
  the installed toolchain requires Node 20.19 or newer (`node:util.styleText`).
  This is a host-tool version limitation, not a TypeScript compilation error.
- Focused Yocto recipes `meter-time`, `meter-time-sync`, `linuxptp`, and
  `msap1-dma`: pass.
- Source-stage `git diff --check`: clean across the five implementation
  repositories.

### 18.4 Final device acceptance

The final check ran on the MNCOS MSAP1 main image built
2026-09-02 13:29:25 UTC, build hash `53fa8c`.

Observed platform state:

- `/dev/meter-time` registered successfully at `0xB00A0000`;
- NTP was the selected/default clock source;
- PTP packages and services were installed but inactive until selected;
- the final inspected result was sequence 128, 59.997 Hz, synchronized and
  Class A qualified, with 143,354 ns uncertainty;
- the result reported zero observer drops and zero rejected cycles;
- eight consecutive live intervals were valid;
- the latest fourteen persisted `seconds_10` intervals were valid;
- one older isolated invalid interval existed in a 22-record inspection
  window, but the scalar history query did not retain enough detail to prove
  its original reason; and
- `seconds_10` was current with 5,511 blocks, about 2.95 MB storage, and a
  three-record historian lag during the live query.

After approximately 150 million ADC frames, the inspected health counters
remained zero for DMA overrun, invalid meter records, sequence gaps, FIFO
overflow, header error, SPI error, RPU CRC error, ring overflow, and
input/output drops.

The separate historian warning found during this check concerns missing
historian routing for valid `FLICKER-v1` and `MAINS-SIGNAL-v1` records. It does
not affect M20 or `seconds_10`; follow-up is recorded in:

```text
applications/MSAP1_APU/docs/future_implementation/
FLICKER_MAINS_SIGNAL_HISTORIAN_ROUTING.md
```

## 19. Known limits and operational guidance

1. The certified M20 profile is 128 kSPS, CH6, filter profile 1, calibration
   profile 1. Other profiles remain visible but produce rejected results.
2. The 125 µs PL elasticity term is conservative and dominates the stable
   uncertainty floor. Reducing it requires a proven architectural change to
   the latch location/FIFO bound, not a UI adjustment.
3. NTP phase corrections can temporarily raise uncertainty. This is expected;
   use PTP only where the network and grandmaster are engineered for it.
4. Immediately after capture start or a rate/configuration change, expect at
   least one warm-up/rate-invalid interval before measured-rate qualification.
5. Rejected intervals must not be replaced by the last valid numeric value.
   Consumers should preserve the sequence and rejection state.
6. PL, RPU, device tree, kernel driver, APU, and Web changes form one release
   set. Partial deployment is unsupported.

## 20. Source implementation map

### 20.1 PL

| Path | Responsibility |
| --- | --- |
| `SourceData/DesignFile/MeterProcessing/meter_frequency_10s_pkg.vhd` | Private contract, boundary tuple, status/reason definitions |
| `.../meter_frequency_10s_conditioner.vhd` | Certified fixed conditioner and group-delay compensation |
| `.../meter_frequency_10s_observer.vhd` | Boundary capture, interpolation, guarded crossings, diagnostics |
| `SourceData/DesignFile/MeterCore/meter_time_control_axi_regs.vhd` | Dedicated `MTC1` AXI4-Lite endpoint |
| `SourceData/DesignFile/MeterCore/meter_core.vhd` | Observer, packetizer, arbiter, and endpoint integration |
| `SourceData/BlockDesign/TopDesign/TopDesign.bd` | M10 AXI route and address assignment |
| `.../tb/meter_frequency_10s_transport_tb.vhd` | Transport/observer verification |
| `.../tb/meter_time_control_regs_tb.vhd` | Register/latch/commit verification |

### 20.2 RPU

| Path | Responsibility |
| --- | --- |
| `R5c1/src/MainApp/aggregation/frequency_10s_protocol.hpp` | Exact `FRQ1` contract |
| `.../frequency_10s_frame_decoder.cpp` | Strict private-frame validation |
| `.../frequency_10s_engine.cpp` | Complete-cycle calculation, validity, records, placeholders |
| `.../aggregation_shadow_service.cpp` | Packet dispatch and continuity integration |

### 20.3 APU

| Path | Responsibility |
| --- | --- |
| `apps/MeterCore/Services/acquisition/support/meter_time_control.*` | `/dev/meter-time` RAII and ioctl adapter |
| `.../pipeline/capture_coordinator.*` | Correlation, uncertainty, rate measurement, boundary scheduling |
| `.../support/utc_clock.hpp` | Read-only `adjtimex` qualification |
| `common/msap1/meter/meter_record.hpp` | Public fixed record contract |
| `common/msap1/meter/meter_data.*` | Strict decoder and typed metadata |
| `.../pipeline/record_ingestor.*` | Independent continuity and durable-before-latest publication |
| `common/msap1/meter/history/meter_history.*` | `seconds_10` projection/query |
| `apps/MeterCore/Services/web-backend/api/meter_*` | REST DTO and route |
| `common/msap1/settings/definition/time_settings.hpp` | NTP/PTP setting |
| `apps/MeterCore/Services/settings/apply/settings_apply.cpp` | Runtime clock-service transition |

### 20.4 Web and Yocto

| Path | Responsibility |
| --- | --- |
| `MSAP1_WEB/src/reading/Frequency10sPanel.tsx` | Operator and developer presentations |
| `MSAP1_WEB/src/reading/ReadingPage.tsx` | Data interval selector and operator context |
| `MSAP1_WEB/src/App.tsx` | Data/Developer navigation, polling, and sync selection |
| `meta-msap1/recipes-kernel/meter-time/` | Kernel module and UAPI |
| `meta-msap1/conf/dtsi/msap1-meter-dma.dtsi` | Dedicated device-tree node |
| `meta-msap1/recipes-support/meter-time-sync/` | NTP polling profile |
| `meta-msap1/recipes-connectivity/linuxptp/` | PTP package configuration |

## 21. Completion statement

M20 implements an end-to-end UTC ten-second frequency product with one
sample-accurate observation path, one arithmetic authority, an isolated time
control endpoint, explicit uncertainty, strict rejection semantics, durable
history, and separate operator/developer presentations.

The final target state demonstrates stable, consecutive Class A-qualified
results. The approximately 129 µs steady uncertainty is consistent with the
documented conservative 16-frame PL elasticity bound, and transient clock
estimate movement is contained by the per-interval 1 ms validity gate.
