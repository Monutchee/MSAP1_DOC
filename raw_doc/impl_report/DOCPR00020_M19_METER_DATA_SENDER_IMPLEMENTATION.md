# MSAP1 M19 Meter Data Sender implementation report

| Field | Value |
| --- | --- |
| Document ID | DOCPR00020 |
| Milestone | M19 Meter Data Sender, reusable Datalogger, canonical meter attributes, and reliable outbound delivery |
| Report date | 2026-09-01 |
| Status | Post-implementation report with host, image, JTAG, and target acceptance evidence |
| Product baseline | Default ADC rate 128 kSPS; 50 Hz and 60 Hz measurement periods |
| Scope | Canonical attributes, historian projection, reusable C++ Datalogger, JSON/CSV writers, HTTP/HTTPS/FTP/SFTP channels, durable outbox, settings and secrets, service IPC, REST API, Web UI, packaging, and target validation |

## 1. Purpose

M19 gives an operator a simple way to turn retained meter history into
scheduled JSON or CSV files and either keep those files locally or send each
file to one or more remote destinations.

The implementation is designed around four product requirements:

- a user selects data by friendly name, measurement period, and calculation;
- the same canonical attribute model is reused by APU APIs, MQTT, History, and
  the Web UI;
- generated files survive service restarts, network outages, and JTAG
  redeployment until delivery is durably acknowledged; and
- the product-neutral generation classes can be reused by another Monutchee
  product without importing MSAP1 services, WebEngine, DMA, RPMsg, or historian
  storage internals.

M19 is an APU, Web, and packaging feature. It reads already persisted,
typed historian data. It does not move samples through RPMsg, access DMA, open
the historian SQLite files directly, or change the real-time metrology
ownership model.

## 2. Milestone outcome

### 2.1 Delivered feature summary

| Feature | Final implementation | User-visible result |
| --- | --- | --- |
| Meter attribute framework | One stable catalog with labels, groups, units, search aliases, value kinds, calculations, and period/access capabilities | The same searchable attribute names appear in Data Logging, History, and MQTT |
| Reusable Datalogger | Product-neutral interfaces and aggregation engine under <code>common/mnc/datalogger</code> | Other products can inject their own historical source, identity, storage, clock, and channels |
| MSAP1 historian adapter | <code>Msap1Datalogger</code> and a typed historian IPC data source | Data Sender never opens historian storage directly |
| File generation | Dependency-injected JSON and RFC 4180 CSV writer strategies | One deterministic file per completed UTC window |
| Data Channels | HTTP, HTTPS, FTP, and SFTP through libcurl/libssh2 | Reusable remote destinations with protocol-specific security and credentials |
| Reliable delivery | SQLite manifest, immutable payloads, independent per-channel state, persisted retry timing | Network failures do not lose files or resend already acknowledged siblings |
| Local-only mode | Atomic archive path with no delivery records | The user can retain files on the meter without any network connection |
| Configuration | Typed jobs, channels, storage policy, validation, and protected material authority | Configuration is guided and invalid combinations are rejected before save |
| Web UI | Logging Jobs, Data Channels, Status, Generated Files, and separate Management cleanup | The complete workflow is available without editing JSON by hand |
| Operations | Health, queue/archive status, preview, authenticated download, retry, and confirmed deletion | Operators can diagnose and manage retained files safely |
| Packaging | Dedicated service identity, systemd unit, tmpfiles, curl SFTP support, and service hardening | Data Sender starts with the product image and recovers queued work automatically |

### 2.2 User-facing design decisions

The final product behavior is:

- Jobs are configured independently from Data Channels.
- A job chooses exactly one historian source period.
- A job chooses a generation interval and a row interval. The generation
  interval must divide into whole rows.
- A job chooses one or more attribute/calculation pairs.
- A job chooses one output format: JSON or CSV.
- A remote job can select multiple Data Channels.
- Local-only is a mutually exclusive destination and performs no send.
- New jobs begin at the next complete UTC boundary.
- Existing queued files retain the job revision and channel selection that
  created them.
- Disabling a job stops future generation but does not discard or stop queued
  delivery work.
- No unsent file is automatically dropped to make room.

### 2.3 Deliberate non-features

M19 does not implement:

- harmonic-spectrum, PQ-event, or waveform export;
- COMTRADE or PQDIF conversion;
- manual arbitrary historical-range export;
- a different content format for each channel of one job;
- compression or file encryption;
- FTPS or cloud-vendor-specific SDKs;
- inbound file reception;
- arbitrary scripts or user-defined templates;
- tariff or billing calculations; or
- an exactly-once delivery claim.

HTTP and FTP remain available for controlled legacy integrations, but they are
explicitly marked as unencrypted and require an operator acknowledgement.

## 3. Release provenance and integration set

The accepted work is published from <code>feat/m19_data_sender</code> into
<code>main</code> through five pull requests:

| Repository | Pull request | Accepted tip | Responsibility |
| --- | --- | --- | --- |
| <code>MSAP1_APU</code> | #58 | <code>f633b6109c53be4cf418e6c46da1cd574254c15e</code> | Canonical catalog, historian projection, reusable Datalogger, Data Sender service, outbox, channels, API, tests, and current-wiring APU integration |
| <code>MSAP1_WEB</code> | #31 | <code>d4ec4005a307cfd951183b23276b109fa98de810</code> | Data Logging workflow, shared attribute picker, Generated Files, Management controls, tests, and current-wiring UI |
| <code>MSAP1_PL</code> | #38 | <code>ee429fecd0d282a912c9961b65aba6db77309b55</code> | Companion ADC current-wiring and DRDY-validation fixes carried by the accepted image |
| <code>MSAP1_RPU</code> | #27 | <code>b25b25dd7002b821b3f722271113bb98d5839fd8</code> | Companion transactional current-wiring apply and aggregation-capacity fixes |
| <code>meta-monutchee</code> | #60 | <code>62baea97b78345ef8f4c1e12d379fe76cc6ca1b8</code> | Service account, systemd, tmpfiles, libcurl/libssh2, package, and image integration |

The PL and RPU changes above are companion ADC/acquisition corrections. They
are part of the five-repository integration set and deployed image, but they
are not dependencies of the Meter Data Sender data path. M19 itself continues
to consume typed historian records and has no PL, RPU, RPMsg, DMA, or ADC
payload interface.

The final software contracts are:

| Contract | Version | Purpose |
| --- | ---: | --- |
| Generated dataset | <code>mnc.meter.datalog.v1</code> | Stable JSON/CSV semantic content |
| Data Sender IPC | 1 | Status, artifact, preview/download, retry, deletion, and channel-test commands |
| Settings IPC | 4 | Active settings plus channel-scoped secret and asset resolution |
| Product settings schema | 6 | M19 Data Logging plus the companion current-wiring configuration |
| Attribute API | API v1 | Snapshot/historian catalog and period-aware capabilities |

Schema 5 introduced Data Logging. Schema 6 subsequently added the companion
current-wiring settings. Migration preserves older M18 settings and creates an
empty M19 configuration, so an upgrade never starts outbound traffic by
itself.

## 4. End-to-end architecture

### 4.1 Component and data flow

~~~mermaid
flowchart LR
    subgraph USER["Operator experience"]
        CONFIG["Configuration<br/>Logging Jobs, Data Channels, Status"]
        HISTORY["History<br/>Generated Files"]
        MANAGEMENT["Management<br/>Independent cleanup"]
    end

    subgraph WEB["Authenticated Web layer"]
        API["MSAP1 Web backend<br/>role-checked API v1"]
        ATTRAPI["Canonical attribute API<br/>snapshot or historian"]
        GATEWAY["Data Sender IPC gateway"]
    end

    subgraph SETTINGS["Configuration authority"]
        ACTIVE["Public active settings<br/>jobs, channels, storage"]
        MATERIALS["Protected channel material<br/>secrets, CA, keys, known hosts"]
    end

    subgraph HISTORY_SERVICE["Historian"]
        STREAM["Durable meter stream"]
        PROJECTION["Typed scalar projections<br/>period, time, quality, sequence"]
        HIPC["Bounded historian IPC query"]
        STREAM --> PROJECTION --> HIPC
    end

    subgraph SENDER["msap1-data-sender"]
        ADAPTER["Msap1HistorianDataSource<br/>product adapter"]
        LOGGER["Reusable Datalogger<br/>UTC rows and calculations"]
        FACTORY["Injected writer factory"]
        JSON["JSON strategy"]
        CSV["CSV strategy"]
        ENGINE["UTC scheduler and retry engine"]
        OUTBOX["SQLite manifest<br/>immutable outbox payloads"]
        ARCHIVE["Local-only archive"]

        HIPC --> ADAPTER --> LOGGER
        ENGINE --> LOGGER --> FACTORY
        FACTORY --> JSON --> OUTBOX
        FACTORY --> CSV --> OUTBOX
        OUTBOX --> ARCHIVE
    end

    subgraph CHANNELS["Independent outbound channels"]
        HTTP["HTTP POST"]
        HTTPS["Verified HTTPS / mTLS"]
        FTP["FTP temp upload + rename"]
        SFTP["SFTP verified host + rename"]
    end

    CONFIG --> API
    HISTORY --> API
    MANAGEMENT --> API
    API --> ATTRAPI
    API --> ACTIVE
    API --> GATEWAY
    ACTIVE -.->|"validated configuration"| ENGINE
    MATERIALS -.->|"runtime-only resolution"| ENGINE
    GATEWAY <--> OUTBOX
    OUTBOX --> HTTP
    OUTBOX --> HTTPS
    OUTBOX --> FTP
    OUTBOX --> SFTP
~~~

The acquisition path ends at the durable meter stream and historian. Data
Sender is a downstream consumer. A slow network, unavailable receiver, full
outbox, or failed sender service cannot backpressure FPGA acquisition or the
meter-record stream.

### 4.2 Ownership boundaries

| Function | Owner | Boundary |
| --- | --- | --- |
| Meter acquisition and record production | PL, R5, and acquisition services | Unchanged by M19 |
| Durable typed meter history | <code>msap1-meter-historian</code> | Data Sender uses historian IPC only |
| Attribute identity and capability metadata | <code>common/mnc/MeterDataProvider/attributes</code> | Shared by snapshot and historian consumers |
| Dataset aggregation and serialization | <code>mnc::datalogger</code> | Product-neutral reusable C++ library |
| MSAP1 source mapping | <code>msap1::datalogger</code> | Adapter from historian IPC to neutral samples |
| Scheduling, outbox, and transfers | <code>msap1-data-sender</code> plus reusable engine | Separate from Web request threads |
| Persistent public configuration and protected material | <code>msap1-settings</code> | Secrets are never returned as settings JSON |
| External API and authorization | <code>msap1-web-backend</code> | All browser access is authenticated |
| Product workflow | <code>MSAP1_WEB</code> | UI consumes capabilities instead of duplicating attribute tables |

## 5. Operator workflow

The UI follows the order in which an operator naturally configures the
feature.

### 5.1 Create a Data Channel

1. Open Configuration → Data Logging → Data Channels.
2. Add a channel and choose HTTP, HTTPS, FTP, or SFTP.
3. Enter only the fields relevant to that protocol.
4. Acknowledge the warning when choosing clear-text HTTP or FTP.
5. Save the public channel configuration.
6. Add the protected credential or verification material after the channel
   has a stable saved ID.
7. Run “Test saved channel” when desired.

Saving a channel never contacts the endpoint. Testing sends a clearly marked
zero-data probe. FTP and SFTP probes remove the temporary test object after
the remote rename test.

### 5.2 Create a Logging Job

The job editor is a four-step review:

1. Name and schedule:
   choose enabled state, source period, generation interval, and row interval.
2. Select data and calculations:
   search the canonical attribute catalog and choose only calculations valid
   for each value kind.
3. Format and destination:
   select JSON or CSV, then Local-only or one or more remote channels.
4. Review:
   read the resulting file cadence, row count, column count, and destination
   summary before saving.

The canonical example is:

| Setting | Value |
| --- | --- |
| Source | Basic 10/12-cycle data |
| Generation interval | 5 minutes |
| Row interval | 1 minute |
| Calculations | Minimum, maximum, and average for selected linear values |
| Result | One file containing exactly five ordered UTC rows |

### 5.3 Monitor and manage

The Status tab shows:

- service health;
- waiting and blocked delivery counts;
- outbox and local archive usage;
- oldest waiting file;
- filesystem free-space guard;
- each job’s next and last window; and
- each channel’s readiness and last test result.

History → Generated Files supports job, date, and state filters, bounded
preview, authenticated download, per-channel delivery detail, manual retry,
and selected deletion.

## 6. Canonical meter attribute framework

### 6.1 One identity, two capability views

<code>MeterAttributeDescriptor</code> is the single source of truth for:

- stable C++ identity and stable external key;
- friendly label;
- group;
- unit;
- search aliases;
- semantic value kind;
- supported calculations; and
- period-specific snapshot/historian capability.

The API exposes this catalog through:

~~~text
GET /api/v1/meter/attributes?usage=snapshot
GET /api/v1/meter/attributes?usage=historian
~~~

Snapshot and historian are filtered views of the same descriptors. They are
not separately maintained key lists.

### 6.2 Measurement periods

| Stable period ID | Label | Snapshot | Historian/Data Sender |
| --- | --- | ---: | ---: |
| <code>basic</code> | Basic (10/12 cycles) | yes | yes |
| <code>cycles_150_180</code> | 150/180 cycles | yes | yes |
| <code>minutes_10</code> | 10 minutes | yes | yes |
| <code>hours_2</code> | 2 hours | yes | yes |
| <code>demand</code> | Demand | yes | yes |
| <code>minutes_10_live</code> | Open 10-minute preview | yes | no |
| <code>hours_2_live</code> | Open 2-hour preview | yes | no |

Frequency is available only from Basic data. Historian energy is selected from
the 10-minute projection, and demand values use the configured Demand period.
Open-interval previews are intentionally excluded from Data Sender because
they can change after publication.

### 6.3 Value-kind-aware calculations

| Value kind | Allowed calculations | Reason |
| --- | --- | --- |
| Linear | minimum, maximum, average, last | Ordinary gauges and signed values |
| Circular angle | circular average, last | A linear average is wrong across 0/360 degrees |
| Cumulative counter | first, last, delta | Exact integer counter behavior and reset detection |
| Peak | last | A published peak already represents its own interval policy |
| Categorical | last | Codes must not be numerically averaged |

The framework preserves existing IDs and appends new scalar identities. M19
completed the public scalar catalog for fundamental values, crest factors,
phase/sequence angles, fundamental active power, and load nature. Indexed
harmonic data remains outside this milestone.

### 6.4 Shared Web selection

<code>AttributePicker</code> is reused by Data Logging and the existing
attribute-driven product pages. It:

- filters by the selected period;
- searches label, key, group, unit, and aliases;
- groups related measurements;
- shows a selected count;
- supports Select visible and Clear; and
- warns when a period change invalidates an existing selection.

The UI requires confirmation before removing period-incompatible selections.
It never silently submits an unsupported attribute.

## 7. Reusable C++ object model

### 7.1 Interfaces and concrete classes

| Abstract boundary | Concrete M19 implementation | Responsibility |
| --- | --- | --- |
| <code>Datalogger</code> | <code>Msap1Datalogger</code> | Generate one typed dataset for a completed window |
| <code>HistoricalMeterDataSource</code> | <code>Msap1HistorianDataSource</code> | Query and page typed MSAP1 historian points |
| <code>MeterDataContentWriter</code> | <code>JsonMeterDataContentWriter</code>, <code>CsvMeterDataContentWriter</code> | Serialize the same ordered dataset |
| <code>MeterDataContentWriterFactory</code> | <code>DefaultMeterDataContentWriterFactory</code> | Inject the chosen writer strategy |
| <code>DataChannel</code> | <code>ConfiguredDataChannel</code> | Turn an artifact into a protocol delivery plan |
| <code>TransferClient</code> | <code>CurlTransferClient</code> | Execute HTTP, HTTPS, FTP, or SFTP I/O |
| <code>OutboxRepository</code> | <code>SqliteOutboxRepository</code> | Persist immutable artifacts, delivery state, and watermarks |
| <code>Clock</code> | <code>SystemClock</code> | Supply UTC time without hard-coding time in algorithms |

<code>HistoricalDatasetGenerator</code> contains the reusable bucket and
calculation logic. <code>DataSenderEngine</code> composes the Datalogger,
writer factory, outbox, and clock into deterministic scheduling and retry
behavior.

### 7.2 Design patterns

| Pattern | Use in M19 |
| --- | --- |
| Dependency inversion | High-level generation depends on interfaces rather than MSAP1 historian or libcurl details |
| Strategy | JSON/CSV writers and protocol channels are interchangeable behaviors |
| Factory | The writer factory selects a serialization strategy without format branches in the scheduler |
| Adapter | <code>Msap1HistorianDataSource</code> converts historian IPC results into product-neutral samples |
| Repository | <code>OutboxRepository</code> hides SQLite and filesystem transaction details |
| Immutable snapshot | Every artifact records the job revision and channel selection used at generation time |
| Value objects | UTC windows, samples, columns, cells, rows, delivery results, and storage policy carry explicit semantics |
| Service facade | <code>DataSenderService</code> owns process lifecycle, configuration reload, worker threads, IPC, and health |

The dependency direction is intentionally one-way:

~~~text
canonical value types
    -> reusable mnc::datalogger core
    -> MSAP1 historian/settings adapters
    -> msap1-data-sender process
    -> authenticated API and Web UI
~~~

The reusable library has no WebEngine, process, DMA, RPMsg, device-node, or
MSAP1 database dependency.

## 8. UTC generation and calculations

### 8.1 Generation flow

~~~mermaid
flowchart TD
    START["Completed UTC job window"] --> SETTLE["Wait 30 seconds for historian settlement"]
    SETTLE --> GUARD{"Outbox quota and<br/>free-space guard allow generation?"}
    GUARD -- "No" --> PAUSE["Keep watermark and all files<br/>raise critical health"]
    GUARD -- "Yes" --> QUERY["Query typed historian IPC<br/>for selected period and attributes"]
    QUERY --> SOURCE{"Source result"}
    SOURCE -- "Historian unavailable" --> DEFER["Keep watermark unchanged<br/>retry later"]
    SOURCE -- "Window expired from retention" --> EXPIRE["Advance watermark without file<br/>record truthful gap"]
    SOURCE -- "Retained window" --> BUCKET["Partition into UTC-aligned<br/>half-open row windows"]
    BUCKET --> VALID["Use only valid samples<br/>preserve valid zero"]
    VALID --> CALC["Apply value-kind-aware calculations"]
    CALC --> COVERAGE["Attach quality, contributing count,<br/>expected count, completeness, continuity"]
    COVERAGE --> WRITER["Create injected JSON or CSV writer"]
    WRITER --> PUBLISH["Write temporary payload<br/>flush, fsync, atomic rename"]
    PUBLISH --> MANIFEST["Commit immutable manifest,<br/>delivery rows, and watermark"]
~~~

All windows are half-open: <code>[start, end)</code>. This prevents a sample
on an exact boundary from appearing in two rows.

The row interval must be compatible with the source period, and:

~~~text
generation_interval % row_interval == 0
~~~

### 8.2 Calculation behavior

Only samples with <code>ReadingQuality::Valid</code> contribute. A value of
zero remains a valid contribution and is never confused with missing data.

Each output cell contains:

- optional exact decimal value;
- quality;
- contributing sample count;
- expected sample count;
- complete/incomplete marker;
- continuity marker; and
- reset epoch where applicable.

Linear averages use overflow-safe accumulation and locale-independent decimal
rendering. Circular averages operate across the 0/360-degree seam. Energy
first/last/delta remains signed 64-bit throughout; a reset-epoch change marks
delta unavailable and continuity false.

No-data and partial-data rows are explicit. The sender never fabricates zero
or silently omits the row.

### 8.3 Historian paging and retention

Historian continuation uses the complete ordering key: measurement timestamp,
block source sequence, record kind, block ID, and attribute identity. A page
ending inside one timestamp or record family cannot duplicate or omit the
remaining values.

If the service is temporarily unavailable, the window remains pending. If the
requested window is older than the retained historian boundary, the scheduler
advances over that expired window without generating a false file and
continues with the first retained window.

## 9. JSON and CSV content contract

Both writers serialize one <code>GeneratedDataset</code>. They do not perform
their own historian queries or calculations.

### 9.1 Common semantics

Every artifact includes:

- schema name;
- deterministic artifact ID;
- job ID and immutable revision;
- product and device identity;
- source period;
- generated time;
- artifact and row UTC windows;
- ordered column identity, label, unit, calculation, and value kind;
- exact value text;
- quality and sample coverage;
- continuity and reset-epoch information; and
- SHA-256 provenance in the manifest and delivery headers.

Artifact identity and filename derive from job ID, revision, UTC start/end, and
format. Operator labels are never used as filesystem identities.

### 9.2 JSON

JSON uses:

~~~text
Content-Type: application/json
schema: mnc.meter.datalog.v1
~~~

Large integers and timestamps are emitted as decimal strings so a downstream
JavaScript consumer cannot lose precision.

### 9.3 CSV

CSV uses UTF-8 RFC 4180 formatting:

~~~text
Content-Type: text/csv; charset=utf-8
~~~

One header describes artifact, row, and repeated cell metadata. Each row
contains the same quality, coverage, continuity, and reset information as the
JSON representation.

### 9.4 Crash-safe publication

Payload publication follows:

1. create an exclusive temporary file;
2. write the complete body;
3. flush and <code>fsync</code> the file;
4. atomically rename it to the deterministic final filename; and
5. <code>fsync</code> the parent directory.

A crash exposes either no final artifact or one complete final artifact.

## 10. Data Channels

### 10.1 Protocol and security matrix

| Protocol | Authentication | Verification and delivery |
| --- | --- | --- |
| HTTP | none, Basic, Bearer | Raw POST; explicit unencrypted-transport acknowledgement |
| HTTPS | none, Basic, Bearer, mTLS | Mandatory peer and hostname verification; system or uploaded CA; optional client certificate/key |
| FTP | username/password | Passive upload to hidden temporary name, remote rename; explicit unencrypted-transport acknowledgement |
| SFTP | password or private key | Required pinned known-host verification; temporary upload and atomic rename |

There is no production option to disable all HTTPS verification. A configured
SFTP channel is not ready until known-host material is present.

### 10.2 HTTP identity and response handling

HTTP and HTTPS POSTs include:

- content MIME type;
- <code>X-MNC-Filename</code>;
- <code>X-MNC-Artifact-ID</code>;
- <code>X-MNC-SHA256</code>;
- stable <code>Idempotency-Key</code>; and
- <code>X-MNC-Zero-Data-Probe</code>.

Response classification is:

| Result | Delivery state |
| --- | --- |
| Any HTTP 2xx | succeeded |
| HTTP 408, 429, or 5xx | retryable |
| Other HTTP 4xx | blocked |
| DNS, connection, timeout, send, receive, or transient TLS connection failure | retryable |
| Authentication, certificate, host-key, unsupported protocol, or invalid configuration | blocked |

FTP and SFTP use a deterministic final filename for receiver-side
deduplication. HTTP receivers must deduplicate the stable idempotency key if a
request was committed but the response was lost.

## 11. Durable outbox and retry model

### 11.1 Persistent layout

The service owns:

~~~text
/data/mnc/data-sender/
├── manifest.sqlite3
├── outbox/
└── archive/
~~~

The SQLite database uses WAL mode and full synchronous commits. It stores:

- immutable artifact identity and metadata;
- safe relative filename, size, MIME type, and checksum;
- source window and generation time;
- immutable job snapshot/revision;
- one independent delivery row per selected channel;
- per-job generation watermark; and
- administrative discard audit records.

Only safe relative filenames are stored. All filesystem operations resolve
under the owned root.

### 11.2 Delivery state flow

~~~mermaid
flowchart LR
    ENQUEUE["Artifact committed"] --> PENDING["pending"]
    PENDING --> CLAIM["in flight<br/>durable lease"]
    RETRY["retry wait<br/>persisted next attempt"] --> CLAIM
    BLOCKED["blocked<br/>operator action required"] -->|"credential or endpoint fixed<br/>or manual retry"| PENDING
    CLAIM --> RESULT{"Transfer result"}
    RESULT -- "retryable" --> RETRY
    RESULT -- "configuration/auth failure" --> BLOCKED
    RESULT -- "acknowledged" --> SUCCESS["succeeded"]
    SUCCESS --> ALL{"Every selected channel<br/>succeeded?"}
    ALL -- "No" --> RETAIN["Keep immutable payload<br/>retry only failed siblings"]
    RETAIN --> RETRY
    ALL -- "Yes" --> COMPLETE["Durably complete<br/>remove remote payload"]
    ENQUEUE -->|"Local-only"| LOCAL["archive<br/>no delivery rows"]
    LOCAL --> ADMIN["Retain until administrator deletes"]
~~~

Successful channels are not resent when a sibling fails. A remote artifact
becomes complete only after every selected channel has a durable success
record.

### 11.3 Retry timing and concurrency

Retry timing is persisted exponential backoff:

- first retry begins at 5 seconds;
- the delay grows to a maximum of 1 hour; and
- stable per-artifact/channel jitter prevents synchronized reconnect storms.

Restart does not reset the attempt count or next-attempt time. The service uses
two delivery workers globally and a per-channel mutex, keeping blocking
network work outside the scheduler/watchdog thread and preventing concurrent
use of one channel runtime.

### 11.4 Local-only behavior

A Local-only artifact:

- is published directly under <code>archive</code>;
- has no channel delivery rows;
- triggers no network connection;
- remains available for list, preview, and authenticated download; and
- is removed only by an explicit administrator action.

### 11.5 Storage protection and recovery

Factory storage policy is:

| Guard | Default |
| --- | ---: |
| Combined outbox/archive limit | 512 MiB |
| Minimum filesystem free space | 256 MiB |
| Completed remote metadata retention | 30 days |

When a guard is reached, generation pauses and health becomes critical.
Existing payloads remain intact and eligible retries continue. There is no
drop-oldest policy.

Startup recovery reconciles:

- temporary files;
- payloads without manifest rows;
- manifest rows without payloads;
- size mismatch;
- unreadable payloads; and
- checksum mismatch.

Missing or damaged payloads become explicit critical state. File absence is
never treated as delivery success.

## 12. Settings and protected material

### 12.1 Public settings

The public <code>data_logging</code> section contains:

- Data Channels;
- Logging Jobs; and
- storage policy.

Validation covers:

- unique and safe IDs;
- supported protocol/authentication pairs;
- scheme, host, port, path, timeout, and traversal rules;
- insecure-transport acknowledgement;
- channel references;
- Local-only exclusivity;
- source period and attribute capability;
- calculation compatibility;
- interval divisibility; and
- selected content format.

Job revision increments whenever generation-relevant configuration changes.
The new revision starts at the next complete boundary; queued old-revision
artifacts are unchanged.

### 12.2 Secrets and assets

Protected material is channel-ID scoped:

- password;
- bearer token;
- private-key passphrase;
- uploaded CA;
- mTLS client certificate and key;
- SFTP private key; and
- SFTP known-host data.

The active settings document, REST responses, generated files, browser
storage, and logs contain presence/status only. The Settings service resolves
the material at runtime only for the dedicated Data Sender service identity.
Runtime asset files are created with owner-only permissions and removed when
the channel is removed or the service stops.

## 13. Data Sender service, IPC, and API

### 13.1 Process model

<code>msap1-data-sender</code> is a dedicated <code>mnc::Service</code>
daemon. It contains:

- one IPC I/O worker;
- one UTC generation worker;
- two bounded delivery workers;
- one settings-change watcher;
- watchdog/readiness integration;
- reload that retains the previous valid configuration on failure; and
- orderly shutdown with runtime asset cleanup.

The service remains installed and running with zero jobs so queued work can
always recover. An empty factory configuration performs no historian query,
credential resolution, DNS lookup, socket connection, or remote write.

The relevant service-manager dependencies are:

~~~text
settings -----------------------> data-sender
meter-stream -> meter-historian -> data-sender
data-sender --------------------> web-backend
~~~

Acquisition readiness does not depend on successful remote delivery.

### 13.2 Internal IPC

The version-1 Unix-stream socket is:

~~~text
/run/monutchee/data-sender/data-sender.sock
~~~

Commands cover:

- status;
- bounded artifact list/detail;
- bounded preview and chunked download;
- selected retry;
- confirmed deletion;
- saved-channel zero-data test; and
- queued-channel-reference validation.

Read/list/download commands are available to peers able to open the
group-restricted socket. Retry, deletion, and channel tests are restricted to
the trusted Web service UID. Channel-reference validation is restricted to
the Settings service UID.

### 13.3 External API and roles

| API | Minimum role | Purpose |
| --- | --- | --- |
| <code>GET /api/v1/meter/attributes</code> | Viewer | Snapshot or historian catalog |
| <code>GET /api/v1/data-logging/configuration</code> | Admin | Jobs, channels, storage, and material presence |
| <code>PUT /api/v1/data-logging/configuration</code> | Admin | Validate and persist Data Logging configuration |
| <code>GET /api/v1/data-logging/status</code> | Viewer | Service, job, channel, queue, archive, and guard status |
| <code>GET /api/v1/data-logging/artifacts</code> | Viewer | Bounded filtered generated-file list |
| <code>GET /api/v1/data-logging/artifact</code> | Viewer | Manifest and per-channel delivery detail |
| <code>GET /api/v1/data-logging/artifacts/preview</code> | Viewer | Bounded text preview |
| <code>GET /api/v1/data-logging/artifacts/download</code> | Viewer | Authenticated manifest-authorized streaming download |
| <code>POST /api/v1/data-logging/artifacts/retry</code> | Admin | Retry selected incomplete deliveries |
| <code>DELETE /api/v1/data-logging/artifacts</code> | Admin | Confirmed selected/all deletion |
| <code>POST /api/v1/data-logging/channels/test</code> | Admin | Saved-channel zero-data probe |
| Credential and asset routes | Admin | Replace/delete secrets and upload/delete verification assets |

List pages, previews, IPC frames, and download chunks are bounded. Artifact
downloads are authorized through the manifest; there is no raw nginx
directory alias.

## 14. Web UI implementation

### 14.1 Configuration → Data Logging

The page has keyboard-accessible tabs:

- Logging Jobs;
- Data Channels; and
- Status.

The channel editor shows an endpoint summary, enabled state, runtime
readiness, last test result, protocol-specific fields, and visible security
warnings. Protected inputs are available only after the channel is saved.

The job editor filters generation and row intervals against the source period
and demand window. It prevents Local-only and remote-channel selection from
being active together.

### 14.2 History → Generated Files

The generated-file view supports:

- job filter;
- delivery-state filter;
- start/end date filter;
- bounded pagination;
- selection;
- preview;
- delivery attempts and state;
- retry;
- protected download; and
- administrator deletion.

The preview explicitly states that quality, coverage, and partial-data markers
are retained as generated.

### 14.3 Management

Management now separates destructive domains:

| Action | Deletes | Preserves |
| --- | --- | --- |
| Clear historian data | Selected/all historian projections | Raw spool, generated files, PQ catalogue, waveforms |
| Clear generated files | Local, completed, and explicitly confirmed unsent artifacts | Historian and raw spool |
| Clear PQ event catalogue | Durable PQ event rows | Waveform files |
| Clear waveform data | Waveform sessions | Historian and generated files |

If any generated file is unsent, the first deletion attempt returns a
conflict. The UI then asks for a second explicit discard-unsent confirmation,
and the outbox records the administrative discard.

## 15. Reliability and failure containment

| Failure | Implemented behavior |
| --- | --- |
| Historian IPC unavailable | Keep the generation watermark; retry later |
| Requested source window expired | Advance over the expired window without fabricating a file |
| One channel succeeds and one fails | Retain payload; retry only the failed channel |
| HTTP 401 or invalid credential | Block and retain until operator correction |
| HTTP 503, timeout, or connection loss | Persist retry-wait with backoff |
| Wrong HTTPS CA or hostname | Block; verification cannot be disabled |
| Wrong SFTP host key | Block; retain payload |
| Service process stops | Manifest, payload, watermark, attempts, and retry time survive |
| JTAG image redeployment | Persistent <code>/data</code> state is reconciled and resumed |
| Storage quota/free-space guard | Pause generation, keep files, continue safe retry, raise critical health |
| Missing or changed payload | Critical missing-payload state; never false success |
| Invalid settings reload | Keep previous valid runtime configuration |
| Acquisition health | Remains independent from sender/network health |

Delivery is at least once. The only unavoidable duplicate boundary is a
receiver that commits a delivery but loses the acknowledgement. Stable HTTP
idempotency keys and deterministic FTP/SFTP final names give receivers a
deduplication identity.

## 16. Packaging and service hardening

The final image adds:

- append-only UID 787 for <code>mnc-data-sender</code>;
- primary <code>mnc-data</code> ownership and only the required
  <code>mnc-settings</code> supplementary access;
- restrictive runtime and persistent directories;
- recursive upgrade-safe ownership repair for persistent data;
- <code>msap1-data-sender.service</code>;
- libcurl, libssh2, and CA certificates; and
- target curl SFTP support.

The unit requires Settings, Meter Historian, and the persistent data mount. It
uses:

- <code>NoNewPrivileges=true</code>;
- <code>ProtectSystem=strict</code>;
- protected home, kernel, and control-group settings;
- only Data Sender persistent/runtime paths as writable;
- only AF_UNIX, AF_INET, and AF_INET6;
- 20-second watchdog;
- restart on failure; and
- restrictive umask.

## 17. Verification strategy

M19 was verified as a ladder. A browser test alone cannot prove fsync/recovery,
and a host unit test cannot prove the target libcurl feature set or persistent
JTAG recovery.

~~~mermaid
flowchart TD
    CONTRACT["Catalog, schema, and interface tests"] --> CORE["Datalogger, writer, scheduler,<br/>outbox, and channel tests"]
    CORE --> APU["Full APU Release build<br/>44 of 44 CTests"]
    CORE --> SAN["Focused ASan and UBSan<br/>10 of 10 tests"]
    APU --> WEB["Web Vitest, TypeScript,<br/>and production Vite bundle"]
    SAN --> ENDPOINTS["Isolated HTTP, mTLS HTTPS,<br/>FTP, and SFTP endpoint matrix"]
    WEB --> YOCTO["Coordinated Yocto image<br/>9740 of 9740 tasks"]
    ENDPOINTS --> YOCTO
    YOCTO --> JTAG["Station-backed JTAG deployment"]
    JTAG --> TARGET["UTC files, multi-channel, outage,<br/>quota, auth, cleanup, secrecy"]
    TARGET --> HEALTH["Zero acquisition impact<br/>and exact settings restore"]
~~~

### 17.1 APU verification

The final Release build completed with warnings enabled. All 44 CTests passed
in 1.00 second. Coverage includes:

- catalog uniqueness and exact capability matrices;
- complete historian scalar projection and paging;
- schema migration and channel/job validation;
- calculation and deterministic writer goldens;
- outbox crash recovery, quota, deletion, and watermark behavior;
- retry classification and sibling-channel independence;
- Data Sender IPC framing and peer authorization;
- REST role, malformed-input, traversal, bound, conflict, and unavailable
  behavior; and
- service lifecycle and target-found regressions.

Ten focused M19, historian, settings, IPC, authorization, catalog, and outbox
tests also passed under ASan/UBSan.

### 17.2 Web verification

The final Web verification completed:

- <code>npm ci</code>;
- 17 of 17 Vitest files;
- 49 of 49 tests;
- <code>tsc -b</code>; and
- the production Vite build.

The resulting main JavaScript bundle was 532.79 kB, 149.67 kB gzip.

### 17.3 Endpoint integration

Isolated loopback tests covered:

- HTTP success, 401, 503, and identity headers;
- mutually authenticated HTTPS;
- trust and hostname failure;
- FTP authentication and final rename failure;
- SFTP password/private-key authentication;
- SFTP known-host verification; and
- atomic temporary-to-final rename.

The live target additionally passed HTTP Basic, HTTP Bearer, verified HTTPS,
FTP password, and SFTP password delivery. The isolated matrix covers mTLS and
private-key SFTP without leaving private fixture material on the product.

### 17.4 Coordinated image build

The accepted command was:

~~~sh
./mnc --cli --from yocto all build
~~~

The final build attempted 9,740 Yocto tasks and all 9,740 succeeded in
00:02:16. The principal transcript is:

~~~text
runtime-generated/buildLog/build_20260831_223147.log
~~~

Accepted artifact identity:

| Artifact | Accepted value |
| --- | --- |
| Provision archive | <code>msap1_yocto_bd2419.tar.gz</code> |
| Provision SHA-256 | <code>bd2419f2053dd4b49494aa9a26ffefe6379d70b4fe50bc51014642414f815e14</code> |
| WIC SHA-256 | <code>1aa1c6b01ccff8e41feb2b211fcca358832d9e848d30de7805b435c239f80ff9</code> |
| Station archive SHA-256 | <code>ef07459bd9db941bee1af10bed8fae3e46349315e45632b556b50d10fc02c934</code> |
| Data Sender executable SHA-256 | <code>0425bdcf9b7e74fe86c529ed3fac8be2f2f229b1a222538e85cd2016cb4e3db8</code> |

## 18. Real-device acceptance

The final accepted Station deployment job was:

~~~text
99605e1f8f7041e311d584e8daa750b9
~~~

It booted target build <code>24b892</code>, built at
<code>2026-09-01 02:32:00 UTC</code>.

### 18.1 Accepted target matrix

| Scenario | Accepted observation |
| --- | --- |
| Canonical JSON job | Basic source, five-minute artifact, exactly five complete one-minute rows |
| Independent CSV job | One header plus five rows with matching semantic content |
| Integrity | Download SHA-256 matched manifest and deterministic filename identity |
| Multiple periods | Basic, 150/180-cycle, 10-minute, 2-hour, energy, and demand historian queries returned typed data |
| Calculations | Linear, circular-angle, reset/delta, partial-quality, and no-data behavior matched deterministic goldens |
| Two remote channels | Successful sibling remained at one attempt while the failed sibling retried 7–8 times |
| Network outage | Six queued payloads and manifests remained byte-identical across JTAG restart |
| Recovery | Queue drained after endpoint repair without resending completed channels |
| Local-only | JSON/CSV created no delivery rows and no receiver traffic |
| Security failures | Wrong CA, hostname, SFTP host key, and bad credentials blocked as designed |
| Retry failures | HTTP 503 and connection drop remained retryable |
| Quota guard | Generation paused, queued bytes stayed constant, retry continued, and health recovered after restoring space |
| Cleanup | Historian and generated-file clears remained independent; unsent discard required explicit confirmation |
| Authorization | Unauthenticated artifact access returned 401; role policy tests passed |
| Secrecy | API, active settings, and journal scans contained no test secret |
| Acquisition impact | DMA, ADC, aggregation, sequence, ring, FIFO, output, and drop counters remained healthy and zero-loss |

### 18.2 Restart behavior

This target is JTAG booted. A Linux <code>reboot</code> cannot bring it back
without another JTAG deployment. Restart acceptance therefore used:

~~~sh
./mnc deploy jtag
~~~

Queue-recovery Station job
<code>fd1b85259be9998c01435374fa560935</code> restored the same image. Six
queued payloads retained their identity, hashes, manifests, and per-channel
states across that cycle.

### 18.3 Final restore

After acceptance:

- the exact original active settings document and content hash were restored;
- M19 jobs and channels were empty;
- test credentials and verification assets were removed;
- generated files and queued bytes were zero;
- receiver certificates, keys, payloads, cookie, and SFTP fixture were
  removed; and
- services were ready with zero stale temporary files.

The restored settings content hash was:

~~~text
fb3a9e849ebba90305a6950750680d1456ab146efd8ff024be74e065c3451c21
~~~

## 19. Target-found release blockers and fixes

Real-device testing found four failures that were not left as operational
workarounds:

| Failure | Risk | Final correction |
| --- | --- | --- |
| An older historian backfill gap disabled retained-window jobs | Valid recent data could never generate | Treat retention truthfully and allow the first fully retained window |
| An expired source window held a durable watermark forever | Catch-up stalled permanently | Advance over expired windows without fabricating artifacts |
| Reload inspected <code>errno</code> after a callback could replace it | Reload could report the wrong wait/signal failure | Preserve the original error before callback work |
| Administrative artifact paging exceeded the repository bound | “Clear all generated files” failed at scale | Share one 500-item maximum between API and repository and add a regression |

Final APU commit <code>f633b61</code> contains focused regressions for all
four cases.

## 20. Known limits and follow-on work

### 20.1 At-least-once boundary

M19 does not claim exactly once. Receivers should deduplicate:

- HTTP/HTTPS by <code>Idempotency-Key</code>; and
- FTP/SFTP by deterministic final filename.

### 20.2 Retention boundary

Data Sender can only export what the historian still retains. Expired windows
are skipped with a visible job error and watermark progress; no synthetic
backfill is created.

### 20.3 Format and export scope

One job emits one selected format. Per-channel formatting and manual
historical-range exports can be added later without changing the core dataset
model. Harmonic, PQ-event, waveform, COMTRADE, and PQDIF exports should use
their own typed source contracts rather than pretending to be scalar
historian rows.

### 20.4 Transport policy

HTTP and FTP are legacy clear-text options. HTTPS or SFTP should be the
default operational choice. A future FTPS requirement would need an explicit
protocol identity, verification policy, endpoint matrix, UI fields, and image
feature gate.

### 20.5 Formal product use

The generated files preserve values, units, quality, coverage, sequence, and
reset provenance. M19 does not itself certify billing accuracy, regulatory
reporting, or IEC conformity.

## 21. Source implementation map

### 21.1 Reusable APU library

| Area | Path |
| --- | --- |
| Canonical attribute model | <code>MSAP1_APU/common/mnc/MeterDataProvider/attributes/meter_attribute.hpp</code> |
| Canonical catalog | <code>MSAP1_APU/common/mnc/MeterDataProvider/attributes/meter_attribute_catalog.cpp</code> |
| Reusable architecture guide | <code>MSAP1_APU/common/mnc/datalogger/README.md</code> |
| Dataset value types | <code>MSAP1_APU/common/mnc/datalogger/types.hpp</code> |
| Datalogger interface and generator | <code>MSAP1_APU/common/mnc/datalogger/datalogger.hpp</code> |
| Calculations | <code>MSAP1_APU/common/mnc/datalogger/datalogger.cpp</code> |
| Content writer strategies | <code>MSAP1_APU/common/mnc/datalogger/meter_data_content_writer.cpp</code> |
| Channel contract and classification | <code>MSAP1_APU/common/mnc/datalogger/data_channel.cpp</code> |
| libcurl transport adapter | <code>MSAP1_APU/common/mnc/datalogger/curl_transfer_client.cpp</code> |
| Outbox repository | <code>MSAP1_APU/common/mnc/datalogger/outbox_repository.cpp</code> |
| Scheduler and retry engine | <code>MSAP1_APU/common/mnc/datalogger/scheduler.cpp</code> |

### 21.2 MSAP1 adapters and service

| Area | Path |
| --- | --- |
| Historian-backed Datalogger | <code>MSAP1_APU/common/msap1/datalogger/msap1_datalogger.cpp</code> |
| Data Sender IPC | <code>MSAP1_APU/common/msap1/datalogger/data_sender_ipc.cpp</code> |
| Data Logging settings | <code>MSAP1_APU/common/msap1/settings/definition/data_logging_settings.cpp</code> |
| Settings secret/asset IPC | <code>MSAP1_APU/common/msap1/settings/settings_ipc.cpp</code> |
| Historian projection/query | <code>MSAP1_APU/common/msap1/meter/history/meter_history.cpp</code> |
| Data Sender process | <code>MSAP1_APU/apps/MeterCore/Services/data-sender/data_sender_service.cpp</code> |
| IPC peer policy | <code>MSAP1_APU/apps/MeterCore/Services/data-sender/ipc_access_policy.hpp</code> |
| Service topology | <code>MSAP1_APU/apps/MeterCore/Services/service-manager/product_units.hpp</code> |

### 21.3 API and Web

| Area | Path |
| --- | --- |
| Attribute API | <code>MSAP1_APU/apps/MeterCore/Services/web-backend/api/attribute_routes.cpp</code> |
| Data Logging API | <code>MSAP1_APU/apps/MeterCore/Services/web-backend/api/data_logging_routes.cpp</code> |
| Data Sender gateway | <code>MSAP1_APU/apps/MeterCore/Services/web-backend/gateway/data_sender_gateway.cpp</code> |
| Authoritative route table | <code>MSAP1_APU/apps/MeterCore/Services/web-backend/api/routes.hpp</code> |
| Shared attribute picker | <code>MSAP1_WEB/src/components/AttributePicker.tsx</code> |
| Data Logging page | <code>MSAP1_WEB/src/configuration/dataLogging/DataLoggingPage.tsx</code> |
| Channel editor | <code>MSAP1_WEB/src/configuration/dataLogging/DataChannelEditor.tsx</code> |
| Job editor | <code>MSAP1_WEB/src/configuration/dataLogging/DataLoggingJobEditor.tsx</code> |
| Status panel | <code>MSAP1_WEB/src/configuration/dataLogging/DataLoggingStatusPanel.tsx</code> |
| Generated Files | <code>MSAP1_WEB/src/history/GeneratedFiles.tsx</code> |
| Management cleanup | <code>MSAP1_WEB/src/management/ManagementPage.tsx</code> |

### 21.4 Packaging

| Area | Path |
| --- | --- |
| Data Sender systemd unit | <code>meta-monutchee/meta-msap1/recipes-support/msap1-apu-app/files/msap1-data-sender.service</code> |
| Persistent/runtime directories | <code>meta-monutchee/meta-msap1/recipes-support/msap1-apu-app/files/msap1-runtime.conf</code> |
| APU package integration | <code>meta-monutchee/meta-msap1/recipes-support/msap1-apu-app/msap1-apu-app_git.bb</code> |
| curl SFTP feature | <code>meta-monutchee/meta-msap1/recipes-support/curl/curl_%.bbappend</code> |
| Stable service identity | <code>meta-monutchee/meta-mncos/conf/include/mnc-identities.inc</code> |

### 21.5 Companion ADC integration

| Area | Path |
| --- | --- |
| PL current mapping | <code>MSAP1_PL/SourceData/DesignFile/AdcConversion/adc_conversion.vhd</code> |
| PL mapping registers | <code>MSAP1_PL/SourceData/DesignFile/AdcConversion/adc_conversion_axi_regs.vhd</code> |
| PL DRDY validation | <code>MSAP1_PL/SourceData/DesignFile/MeterProcessing/meter_spectral_conditioner.vhd</code> |
| R5C0 transactional apply | <code>MSAP1_RPU/R5c0/MainApp/handlers/meter/meter_config.cpp</code> |
| RPU shared metering contract | <code>MSAP1_RPU/common/include/metering.hpp</code> |
| R5C1 capacity policy | <code>MSAP1_RPU/R5c1/src/MainApp/aggregation/aggregation_scheduler_policy.hpp</code> |

### 21.6 Acceptance evidence

| Evidence | Path or identity |
| --- | --- |
| M19 checklist | <code>runtime-generated/doc/impl_checklist/M19_METER_DATA_SENDER_CHECKLIST.md</code> |
| Final build transcript | <code>runtime-generated/buildLog/build_20260831_223147.log</code> |
| Final Station job | <code>99605e1f8f7041e311d584e8daa750b9</code> |
| Final installed build | <code>24b892</code> |

Runtime-generated evidence reconstructs the accepted run. Versioned
interfaces, strict validators, source files, and repository tests remain the
authoritative implementation contract.

## 22. Completion statement

M19 is implemented, committed, built, JTAG-deployed, target-accepted, and
restored.

The completed product lets an operator configure reusable Data Channels,
create UTC-aligned Logging Jobs, select period-valid attributes and
calculations, generate deterministic JSON or CSV, keep files locally, or
deliver each file independently to multiple HTTP, HTTPS, FTP, and SFTP
destinations.

The most important reliability decision is that the durable outbox owns the
artifact before network delivery begins. Unsent data is preserved, successful
siblings are not resent, retries survive restart, storage pressure stops new
generation instead of deleting customer data, and remote failures remain
isolated from acquisition.

The most important maintainability decision is the dependency-inverted
<code>mnc::datalogger</code> library. Another product can reuse the UTC
windowing, calculations, writer strategies, scheduler, outbox, and channel
contracts by supplying its own historical source and product adapters.
