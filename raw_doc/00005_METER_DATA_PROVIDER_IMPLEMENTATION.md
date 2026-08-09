# MSAP1 MeterDataProvider implementation report

## Status and scope

This document records the implementation of the product-neutral
`mnc::meter::MeterSnapshotProvider` interface and the MSAP1 adapter
`msap1::meter::Msap1MeterDataProvider`.

The implementation is a latest-value distribution layer for applications that
need a current reading. It is not a database writer and does not promise
lossless delivery to every subscriber. A future durable path is documented in
`FUTURE_DURABLE_METER_PIPELINE.md`.

## Motivation

The acquisition process owns DMA, record validation, and decoding. Web, CLI,
Modbus, and future publishers should not know the PL packet layout, channel
order, or RPMsg lifecycle. A typed provider supplies one boundary:

- acquisition decodes records into sparse typed updates;
- the shared store owns the latest view for each period;
- the MSAP1 adapter projects selected values and units;
- consumers receive immutable snapshots with timing and quality provenance.

Measurement calculations remain in PL. The APU performs record decoding,
state publication, selection, and transport—not RMS or frequency calculation.

## End-to-end architecture

~~~mermaid
flowchart LR
    ADC["AD7771 / PL meter logic"] --> DMA["AXI DMA S2MM"]
    DMA --> DEV["/dev/msap1-meter"]
    DEV --> ING["msap1-fpga-acquisition\nDMA reader"]
    ING --> DEC["MeterDecoderRegistry\nMTR1/MTR2 decoders"]
    DEC --> UPD["msap1::MeterUpdate\nsparse typed update"]
    UPD --> DATA["msap1::MeterData"]
    DATA --> STORE["MeterLatestStore\nindependent period views"]
    STORE --> ADAPTER["Msap1MeterDataProvider"]
    ADAPTER --> SNAP["mnc::meter::MeterSnapshot"]
    SNAP --> CLIENT["Typed acquisition IPC client"]
    CLIENT --> WEB["Web API / dashboard"]
    CLIENT --> CLI["mnc CLI"]
    CLIENT --> PUB["Future publishers"]

    DEC -. every-record future path .-> DURABLE["MeterRecordPublisher\nDurableMeterSpool\nDatabaseWriter"]
~~~

The dashed path is intentionally not part of the current provider. Latest
consumers may coalesce updates; a historian must use a future durable record
pipeline.

## Library and ownership boundaries

~~~mermaid
classDiagram
    class MeterSnapshotProvider {
        <<interface>>
        +capabilities() vector~MeterCapabilities~
        +latest(request) optional~MeterSnapshot~
        +subscribe_latest(request, callback) LatestSubscription
    }
    class Msap1MeterDataProvider {
        -MeterData& data_
        +capabilities()
        +latest(request)
        +subscribe_latest(request, callback)
        -project(view, request) MeterSnapshot
    }
    class MeterData {
        -MeterLatestStore store_
        +apply(MeterUpdate)
        +latest(period) optional~MeterPeriodView~
        +subscribe(period, callback) Subscription
    }
    class MeterLatestStore {
        +apply(MeterUpdate)
        +latest(period) optional~MeterPeriodView~
    }
    class MeterDecoderRegistry {
        +decode(record) optional~MeterUpdate~
    }
    class MeterUpdate {
        +period MeasurementPeriod
        +kind RecordKind
        +sequence uint64
        +configuration_generation uint32
        +optional typed fields
    }

    MeterSnapshotProvider <|.. Msap1MeterDataProvider
    Msap1MeterDataProvider --> MeterData : references
    MeterData *-- MeterLatestStore
    MeterDecoderRegistry --> MeterUpdate
    MeterUpdate --> MeterData : apply
~~~

`mnc::meter` contains generic value types, the attribute catalogue, request
semantics, and provider interfaces. `msap1::meter` contains MSAP1 decoders,
`MeterData`, and the adapter. The generic library has no WebEngine, CLI, RPMsg,
or DMA dependency.

## Typed data model

The public snapshot contains only requested fields:

~~~cpp
struct MeterAttributeValue {
    MeterAttributeKey attribute;
    MeterUnit unit;
    ReadingQuality quality;
    std::int64_t value;
    std::uint64_t source_sequence;
    std::int64_t measured_at_nanoseconds;
    std::uint32_t sample_count;
    std::int64_t calculation_window_nanoseconds;
};
~~~

Values use integer engineering units:

| Family | Unit | Example |
| --- | --- | --- |
| Frequency | millihertz | `59998` = 59.998 Hz |
| Line-neutral voltage | microvolts | `120000000` = 120 V |
| Current | microamperes | `5000000` = 5 A |

A valid zero is different from unavailable. Quality is per attribute, not a
single status for the whole snapshot. Source sequence, measured time, sample
count, and calculation window are copied from the source period view.

## Attribute catalogue and request semantics

Stable IDs and API keys are defined in
`common/mnc/MeterDataProvider/attributes`:

| ID | External key | Unit | Current Basic status |
| --- | --- | --- | --- |
| `Frequency` | `frequency` | mHz | Produced |
| `VanRms/VbnRms/VcnRms` | `voltage.ln.a/b/c.rms` | µV | Produced |
| `VabRms/VbcRms/VcaRms` | `voltage.ll.ab/bc/ca.rms` | µV | Defined, unavailable |
| `IaRms/IbRms/IcRms/InRms` | `current.a/b/c/n.rms` | µA | Produced where valid |

`MeterAttributeSet` supports ordered, duplicate-free combinations of
`Frequency`, `VoltageLnRms`, `VoltageLlRms`, `CurrentRms`, `Fundamental`, and
`AllDefined`. Empty request selections expand to canonical provider order.
Explicit selections retain caller order after de-duplication.

Known-but-unsupported attributes produce `ReadingQuality::Unavailable`.
Malformed IDs or invalid indexed keys are rejected with
`std::invalid_argument`. Indexed keys are reserved for future harmonic,
flicker, or other indexed families and are not silently treated as scalars.

## Period isolation

The period enum currently defines:

~~~cpp
enum class MeasurementPeriod : std::uint8_t {
    Basic,         // 10/12 cycles
    Cycles150_180, // 15 Basic blocks
    Min10,         // reserved
    Hour2          // reserved
};
~~~

Each period has independent state. A future 1-second, 3-second, 10-minute, or
two-hour record must update its own view. Missing values never inherit from a
different period. The adapter advertises only `Basic` and `Cycles150_180` until
the corresponding producer and decoder are implemented.

## Ingestion and projection workflow

~~~mermaid
sequenceDiagram
    participant PL as PL / AXI DMA
    participant R as DMA reader
    participant D as Decoder registry
    participant M as MeterData
    participant S as MeterLatestStore
    participant P as Msap1 provider
    participant C as Web/CLI

    PL->>R: validated DMA record
    R->>D: bytes and format metadata
    D->>D: validate magic, length, version, sequence
    D-->>R: MeterUpdate
    R->>M: apply(update)
    M->>S: update selected period
    C->>P: latest(request)
    P->>M: latest(period)
    M-->>P: immutable MeterPeriodView
    P-->>C: projected MeterSnapshot
~~~

The decoder registry is the only component that understands MTR1/MTR2 bytes.
It creates a `MeterUpdate` containing period, record kind, sequence,
configuration generation, optional typed value groups, and timing metadata.
`MeterData::apply` updates `MeterLatestStore` and notifies subscribers.

## Msap1MeterDataProvider implementation

The adapter implements:

1. `capabilities()`: returns only currently supported period/attribute
   combinations.
2. `latest(request)`: obtains one period view and calls the common projection
   routine.
3. `subscribe_latest(request, callback)`: subscribes to the underlying period
   and projects each view with the same rules.

Projection is deterministic:

- empty selection expands to canonical order;
- explicit selection preserves request order after duplicate removal;
- supported attributes copy values, units, quality, and provenance;
- known unsupported attributes become unavailable;
- malformed requests throw;
- no source view returns `std::nullopt`.

The subscription owner holds the underlying move-only
`MeterData::Subscription`. The callback captures the request, not the provider,
so it does not dereference a destroyed adapter. Destroying the returned
`LatestSubscription` unregisters the callback.

## Latest-state delivery contract

Subscriptions are coalescing latest-value notifications. A slow consumer can
miss intermediate updates, but cannot stall DMA ingestion or other consumers.
Callbacks must be short and nonblocking; expensive work belongs on a consumer
executor or queue.

This is the correct contract for:

- Web dashboards;
- REST/JSON reads;
- CLI snapshots;
- Modbus register images;
- MQTT/IEC 61850/BACnet “current value” publishers.

It is not the correct contract for a historian. The future historian receives
every accepted record through a separate durable boundary.

## Web and CLI integration

~~~mermaid
flowchart TB
    REQUEST["Web or CLI typed request"] --> IPC["MSAP1 acquisition IPC v14"]
    IPC --> GATE["AcquisitionGateway / typed client"]
    GATE --> CAP["CaptureCoordinator"]
    CAP --> PROVIDER["Msap1MeterDataProvider"]
    PROVIDER --> SNAPSHOT["MeterSnapshot"]
    SNAPSHOT --> DTO["MeterSnapshotResponse / CLI result"]
~~~

The Web backend maps the typed snapshot to the stable
`MeterSnapshotResponse` DTO and JSON representation. CLI commands use the same
typed request and choose human or machine-readable output. Neither application
opens DMA devices or parses PL records.

## Future durable stream

~~~mermaid
flowchart LR
    RECORD["Validated PL record"] --> PUBLISHER["MeterRecordPublisher"]
    PUBLISHER --> SPOOL["DurableMeterSpool"]
    SPOOL --> DB["DatabaseWriter process"]
    SPOOL --> HIST["Historian / replay consumers"]
    LATEST["MeterDataProvider latest state"] -. separate lossy view .-> PUBLISHER
~~~

The current provider does not create or own a SQLite database. This keeps the
prototype's current-value path simple. When the database service is introduced,
it can consume the durable stream without changing the snapshot API.

The planned components and responsibilities are:

- `MeterRecordPublisher`: accepts validated records at the acquisition boundary.
- `DurableMeterSpool`: ordered, replayable storage with consumer cursors.
- `DatabaseWriter`: owns database schema, batching, retention, and database I/O.

## Failure behavior and verification

Expected conditions are explicit:

| Condition | Provider behavior |
| --- | --- |
| No record after startup | `latest()` returns `std::nullopt` |
| Known unsupported field | Snapshot contains `Unavailable` |
| Malformed selector | Reject with `std::invalid_argument` |
| Valid measured zero | Valid quality and value zero |
| Slow latest subscriber | Intermediate views may coalesce |
| DMA/decoder fault | Report through acquisition health; no fabricated value |
| Future durable-storage fault | Stop/raise the durable pipeline fault; do not hide it in latest state |

The implementation should be verified for:

- capability advertisement for Basic and Cycles150_180;
- empty/grouped/explicit selection;
- stable ordering and duplicate removal;
- unsupported and malformed keys;
- valid zero versus unavailable;
- source sequence and timing preservation;
- subscription lifetime and coalescing;
- concurrent Web, CLI, and publisher consumers;
- period isolation when future periods are added.

## Extension procedure

To add a new attribute family:

1. Add a stable ID, API key, unit, and catalogue descriptor.
2. Add typed fields to the MSAP1 value model if required.
3. Extend a PL record and decoder without changing existing record meanings.
4. Apply it only to the intended period view.
5. Add capability advertisement after the producer is operational.
6. Add Web/CLI DTO mappings and tests.

To add a period, add the enum, independent store state, record mapping, decoder,
capability, and tests. Never copy a value from another period merely because
the new period is not available yet.

## Related documentation

- Programming guide:
  `MSAP1_APU/doc/METER_DATA_PROVIDER_PROGRAMMING_GUIDE.md`
- IPC/service architecture:
  `MSAP1_APU/docs/IPC_SERVICE_ARCHITECTURE.md`
- Durable pipeline proposal:
  `MSAP1_APU/common/mnc/MeterDataProvider/FUTURE_DURABLE_METER_PIPELINE.md`

