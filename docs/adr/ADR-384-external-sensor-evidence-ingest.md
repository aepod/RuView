# ADR-384: External sensor ingest: spatial evidence and object readings

- **Status**: proposed
- **Date**: 2026-10-05
- **Deciders**: RuView maintainers
- **Tags**: sensor-hal, ingest, interop, radar, readings, senml, ground-truth, wire-format
- **Related**: ADR-320 (Sensor HAL), ADR-311 (sensor fusion), ADR-306
  (canonical spatial ontology, amended by this ADR), ADR-305 (authenticated
  sensor identity), ADR-301/302 (calibration, domain state), ADR-063 (mmWave
  fusion), ADR-382 (`spatial.evidence.v1` RF export, amended by this ADR).
  Planned: ADR-385 (ESP32 UDP frame contract), ADR-386 (producer packaging as
  signed RVF). Wire format origin: WeftOS ADR-107 §7 and its 2026-10-03
  amendment; SenML, IETF RFC 8428.
- **Plan**: `docs/design/sensor-contracts-plan.md`. Inputs:
  `docs/design/sound-and-radio-ranging.md` and
  `docs/design/snapshot-node-sounding.md` (WeftOS session notes,
  2026-10-06); ADR-110 §A0.11–A0.12 (CSI sync packet).

Numbering: on 2026-10-05 `main` had up to ADR-382, and open PR #2161 claimed 383.

## Context

Every new sensor reaches RuView today through a bespoke path. The ESP32
firmware probes and parses two radar families on UART1 (MR60BHA2/LD6002 at
115200 baud, LD2410 at 256000), and ADR-063 phase 4 plans more (LD2450). Each
addition grows the firmware, requires RuView maintainers to own a parser for a
device they may not have, and needs validation on hardware that CI cannot
reach. #2142 shows the cost: a probe that cannot tell an LD6002B or LD6004 from
an MR60BHA2 decodes the wrong report as a breathing rate.

ADR-320 already rejects "adding per-modality ingest paths". It defines one
`SensorHal` trait and one observation type for fusion, but the trait lives
inside the process. A device outside RuView can implement it only by adding
Rust to this repository. `ruview-hal` ships only synthetic reference adapters,
and the sensing server does not depend on it.

Producers already exist outside RuView. Host-side readers ("cogs") cover
LD6002, LD2450, RD-03E, ToF and other parts. The radar readers write
`spatial.evidence.v1` (`radar_track_point`), either to a file or served at
`GET /spatial?seconds=N`. ADR-382 makes RuView a producer of the same format.
Readings that are not spatial, such as temperature, gas and power, have no
shared format: each reader encodes the unit in a field name (`temp_c`,
`pressure_hpa`).

These inputs are not all the same kind of data. Some measure the **people** in
a space and the geometry they move through. Others report the **state of an
object or space**. The ontology (ADR-306) has spaces, zones and objects, but no
way to say "this reading is property P of entity E".

## Decision

RuView accepts external sensor data through two inbound contracts and fuses
everything it can into one view of a space. It does not add device drivers for
sensors that an external producer can read.

### 1. Scope: where RuView stops

RuView fuses every input it can get into one view of a space: the space's
geometry, the state of its objects, and the people in it, each with
uncertainty and an evidence level.

| In scope | Out of scope (owner) |
|---|---|
| Ingesting any producer that speaks a contract: frames, evidence, readings | Acquisition, drivers and per-device firmware (producers, cogs) |
| The world model: spaces, objects and their state, sensors and their poses | Long-term telemetry history and dashboards (WeftOS, Homecore) |
| People inference (presence, count, position, pose, activity, vitals) and its publication gates | Automation and actuation (Homecore, ADR-128) |
| Declared links from readings to people (cues) and to sensors (covariates) | Rendering: projections and clients read the world model |
| Evidence levels, calibration and qualification of every output | Identifying people beyond anonymous tracks (privacy policy) |

Republishing the world model stays in scope, through MQTT (#2117) and the
`spatial.evidence.v1` export (ADR-382), because that is how projections and
automation receive it.

### 2. Two kinds of input, kept apart

| Kind | What it describes | Contract | What RuView does with it |
|---|---|---|---|
| **Perception evidence** | The occupants of a space, and the geometry they move through | `spatial.evidence.v1` (§3) | Fusion and references for people inference |
| **Object readings** | The state of an object or a space | SenML profile (§4) | Updates that entity's state in the world model. Not evidence about people by default |

A reading becomes relevant to perception only through a declaration. Nothing
is inferred from the sensor type:

- **Interaction cue.** A state change that implies a person acted, such as a
  door opening, a light switching on, a stove drawing power, or CO2 rising in a
  closed room. Each cue is declared per quantity and entity with its own
  likelihood. A cue enters as a corroborating reference first, feeds fusion
  only after its likelihood is measured, and never decides presence alone.
- **Sensor covariate.** A reading that changes how a sensor behaves. For
  example, temperature and humidity shift CSI, and a metal door's state
  changes multipath. Covariates feed calibration and domain-state checks
  (ADR-301/302), never the person estimate.

The two kinds lift into different types: evidence becomes `HalObservation` and
readings become `Reading` (§5). Fusion cannot mistake one for the other.

### 3. Perception evidence: `spatial.evidence.v1`

- **Accepted types:** `radar_track_point` (with the optional `sensor` beam
  block), `radar_range`, `pose`, `shell_measure`, `human_confirm`, `tof_depth`,
  `uwb_echo` and `imu_event`, alongside the existing `rf_gaussian` and
  `rf_link_observation`. Body rules follow WeftOS ADR-107 §7 and §7.1. They are
  listed in the plan, Appendix A.
- **Envelope:** `schema`, `type`, `t_ns`, `frame` (`room_enu` only), `region`,
  `source_id`, `uncertainty_m` (0, 100], and `provenance{receipt, producer,
  proof}`. Ids follow the ADR-382 pattern. Lines are at most 16 KiB.
- **Unknown types** get a dedicated `UnknownType` error and are counted.
- **Crate:** inbound parsing lives in `ruview-spatial-evidence`, so one
  validator serves both directions. The RF export and its `ruview-unified`
  dependency move behind a default `rf-export` feature, so the server can
  depend on the crate without pulling in `ruview-unified`. ADR-382's
  "emitter only" rule is amended accordingly. The emitter still refuses
  non-RF types unless the `ingest` feature is enabled.
- **Vitals are about a person.** Device-reported heart and breathing rates may
  arrive as SenML (§4), but their subject must be an anonymous track or a
  space-level occupant slot, never an `Object`. They stay behind the
  calibrated vitals publication gate (#2147). The wire format grants no
  authority to publish. Until a track binding exists, external vitals are
  stored and counted, but not published or fused.
- **Record types not yet in v1** are rejected and counted as unknown: an
  acoustic range, a cooperative radio delay between scheduled nodes, and a
  clock-correction row. RuView adds them in the same change in which WeftOS
  ADR-107 defines them.

### 4. Object readings: SenML profile

- **Wire:** SenML (RFC 8428), JSON, one pack per line. Each record has a name
  (`n`), a unit (`u`) from the IANA SenML Units registry, a value (`v`, `vb`,
  `vs` or `vd`) and a time (`t`). Base fields (`bn`, `bt`, `bu`) may be used.
- **RuView profile:**
  - `bn` is required and is the source id. It follows the evidence id rule.
  - `proof_` is required: `MEASURED`, `CODE` or `SYNTHETIC`. The trailing
    underscore makes it must-understand under SenML's extension rules, so a
    reader that ignores it must reject the record.
  - `entity` is optional and names the ADR-306 `Space` or `Object` the reading
    describes.
  - `sigma` is optional: 1σ uncertainty in the reading's own unit.
- **Quantity vocabulary:** a versioned file maps each name (`temperature`,
  `humidity`, `co2`, `tvoc`, `pm2_5`, `illuminance`, `sound_level`,
  `contact`, `power`, …) to exactly one SenML unit, so `temperature` is always
  `Cel`. Records with unknown names are stored and counted, and nothing
  consumes them. Cue and covariate declarations (§2) live alongside the
  vocabulary, each with a test vector.
- **Binding table:** configuration maps `source/quantity` to
  `entity.property`, for example `sen-temp-1/temperature →
  space:bedroom.air_temperature`. A record's `entity` field overrides the
  table.
- WeftOS has agreed to this readings contract, so producers and RuView share
  one format.

### 5. World model changes (amends ADR-306)

- **Add a `Reading` fact:** `{ subject: EntityRef, property, value, unit,
  at_unix_ms, evidence_level, provenance }`.
- **Entity state:** `Space` and `Object` gain a bounded current-state map built
  from readings.
- **Evidence levels** come from `proof`: `SYNTHETIC` maps to L0 with the
  synthetic calibration handle, `CODE` to L1, and `MEASURED` to L2. L3 and
  above come only from fusion or calibration, never from the wire. Wire `t_ns`
  becomes `at_unix_ms`.
- **HAL:** an `EvidenceHal` (`SensorHal<Raw = EvidenceRecord>`) lifts
  perception evidence into `HalObservation`. Radar types map to `Mmwave`,
  `uwb_echo` to `Uwb`, `imu_event` to `Imu`, and `tof_depth` to
  `Custom("tof")`. Until a fusion ADR wires `ruview-fusion` into the server,
  this adapter is library-level and tested in its crate.

### 6. Transports

All transports share one validator, one per-source limiter and one set of
counters. The server dispatches on content: a line carrying `schema` is
evidence, and a JSON array is a SenML pack.

| Transport | Shape | Purpose |
|---|---|---|
| **Push** | `POST /api/v1/evidence`, JSONL, body ≤ 1 MiB (route-level limit) | Any producer that can POST |
| **Pull** | `--evidence-pull URL[,…]`: poll a producer's `GET /spatial?seconds=N`; dedupe by `provenance.receipt` | Existing radar readers with no changes |
| **File** | `--evidence-file PATH` (replay) and `--evidence-follow PATH` (tail) | CI, recorded sessions, file-only producers |

A WebSocket transport is deferred. UDP and MQTT ingest are not added. Pull URLs
come only from CLI or environment configuration, never from record content.

### 7. Trust, limits and privacy

- **Mounting:** the ingest routes exist only when `RUVIEW_API_TOKEN` is set.
  They are placed before the bearer-auth layer. Push needs `sensing:admin`
  unless a narrower `evidence:ingest` scope is adopted.
- **Sources:** `--evidence-sources` (`RUVIEW_EVIDENCE_SOURCES`) is the
  allowlist, or ADR-305 enrolment once that lands. An empty list rejects
  everything.
- **Rate limit:** a per-source token bucket, 100 lines/s sustained with a
  burst of 500 by default. Excess lines are dropped and counted, never queued
  without bound.
- **Clock:**
  - **Two times per record:** each record keeps the producer's `t_ns`
    unchanged and the server's receive time.
  - **Freshness:** checked against receive time. A record is rejected when
    its receive time is more than 5 s ahead of the server or older than
    `--evidence-max-age` (default 30 s).
  - **Corrections:** when a producer publishes clock corrections (anchor,
    offset, rate), RuView keeps the latest fit for each source. It computes
    `t_shared = t_local − (offset + rate × (t_local − t_anchor))` and applies
    each new fit back over that source's buffered records. Stored records are
    never rewritten. The correction is a separate fact, and reads return both
    the raw and the corrected time.
  - **Until corrections exist:** the producer time is used as sent, and skew
    is reported per source.
- **Regions:** records are used only for registered regions. A region is
  registered by `--evidence-regions` or by its first `shell_measure`, which
  resolves it to a room shell and frame. Records for an unregistered region
  are validated, stored and counted, but nothing consumes them.
- **Bounds:** positions outside a known `shell_measure` (plus
  `uncertainty_m`) are rejected.
- **Ground truth:** `CODE` and `SYNTHETIC` records never serve as ground truth
  for calibration, qualification or training labels.
- **Privacy mode (#2125):** track positions and object readings are withheld
  from REST and WebSocket and are not recorded. Readings such as stove power,
  door contacts and light reveal routines as clearly as presence does. Only
  counts and presence remain.

### 8. State and exposure

- **`EvidenceStore`:** keeps, per source, the last 30 s or 4096 records,
  whichever is smaller. It also keeps the latest `pose` per node, the latest
  `shell_measure` per region, entity state from readings, and counters
  (accepted, rejected by reason, rate-limited, deduplicated, skew).
- **`GET /api/v1/evidence/status`** returns the counters per source.
- **`GET /api/v1/evidence/latest?seconds=N`** returns recent records, subject
  to privacy rules. `/api/v1/status` gains a short summary.
- **Recordings:** while a recording runs, accepted lines are written to
  `<data-dir>/recordings/<session>.evidence.jsonl` next to the CSI recording.

### 9. First consumers (validation plane only)

- **Labeller:** `ruview_groundtruth::radar::windows_from_evidence` turns
  radar records into `RadarWindow`s, so `auto_label` works on a recording plus
  its evidence file. `verified` is set only for `MEASURED` records.
- **Calibration status:** `GET /api/v1/calibration/status` reports agreement
  between calibrated presence and the evidence references (radar tracks,
  `human_confirm` boxes, and declared cues once measured), with sample counts.
  These are diagnostics only. They never change a decision or a publication
  gate. They supply the occupied controls that qualification lacks (#1952).

### 10. Compatibility

This ADR is additive. The ESP32 UDP frames, the firmware UART1 radar path, and
every existing flag and endpoint keep working unchanged. With no evidence
flags set, the server behaves exactly as before. Nothing is deprecated here.
An existing path is retired only after its replacement has run side by side
with it, matched on recorded data, and a later ADR has decided the retirement.

### 11. Conformance

RuView ships valid and invalid test vectors for every evidence type and
profile rule. The evidence vectors are generated from, and checked against,
the WeftOS ADR-107 examples and the reference validator, so the two cannot
drift silently. It also ships a check tool (`evidence_check FILE`). A producer
conforms when its output passes the check. RuView needs no access to the
device itself.

## Consequences

### Positive

- **New sensors cost RuView no drivers.** The firmware keeps CSI and its
  existing radar path, and gains no new device drivers.
- **One view from all inputs.** People inference, object state and geometry
  share one world model. Each output carries its evidence level.
- **People and objects stay apart.** Object readings cannot silently become
  presence evidence; every link is declared and measured.
- **Placement and room geometry arrive as data** (`pose`, `shell_measure`).
- **Occupied controls for calibration** come from radar tracks, operator
  labels and, later, measured cues.
- **Shared formats.** RuView, WeftOS and the cog producers use the same
  evidence format and the same readings profile.

### Negative

- **Producers run where the sensor is attached.** A producer runs as a native
  binary on any platform or, later, as a signed portable package (ADR-386).
  A sensor wired only to an ESP32 UART still needs the node to read it.
- **Format changes now ripple both ways.** ADR-382's amend-in-the-same-change
  rule extends to this ADR and to the vocabulary file.
- **New attack surface.** Token auth, the allowlist, body and rate limits,
  bounds, typed rejections and counters mitigate it.
- **More model surface.** The `Reading` fact and entity state add to the
  ontology and must stay bounded.

## Phases

| Step | Content | Status |
|---|---|---|
| 1 | Inbound evidence parsing, crate feature split, vectors, check tool | Library only |
| 2 | Server ingest: push, pull, file, store, counters, recording, trust and privacy controls | |
| 2b | Readings: SenML profile, vocabulary, binding table, `Reading` and entity state, dispatch | Cues and covariates declared but not consumed |
| 3 | Labeller adapter, calibration-status diagnostics, `EvidenceHal` | |
| 4 | First external producer | See below |
| 5 | WebSocket transport; the firmware mmWave path as an evidence producer | Later |

**Step 4 detail:** an LD2450 reader serving `GET /spatial`, pulled by the
server, in a measured room, with empty, walking and seated phases and operator
`human_confirm` labels. Results are tagged MEASURED with a reproducer.

## Validation

- **Unit:** every accepted type and profile field round-trips, and every rule
  has a rejecting vector. RuView and the reference validator agree on every
  evidence vector.
- **Integration:**
  - Push, pull (stub server, dedupe) and file replay give identical counts
    across runs.
  - Requests without a token get 401.
  - Oversized bodies get 413.
  - Rate limiting works.
  - Privacy mode suppresses positions and readings.
- **Non-regression:** with no evidence flags, the existing server test suites
  pass unchanged.
- **Hardware (step 4):** the producer's output passes the check tool. Radar
  presence is compared with operator labels, and calibrated presence with
  both. Results are reported with sample counts.
