# Plan: sensor contracts, one step at a time

- **Status:** draft for review, 2026-10-05
- **Base:** upstream `main` @ `0ef6b96f`. Code evidence below was read at `0b15c0eb`; the only change between the two is README docs.
- **Decision records:** ADR-384 (draft, in this branch). ADR-385 is reserved for step 3.
- **Scope:** RuView defines contracts and consumes them. It stops growing device-specific code. Every step is additive; nothing existing is removed or deprecated.

## 1. Why

RuView currently owns the full chain from radio silicon to inference: ESP32 firmware, UART radar drivers, edge vitals, OTA, provisioning and time-sync. Every new sensor adds firmware, a parser and hardware validation that CI cannot reach. #2142 shows the cost. A radar probe that cannot tell an LD6002B from an MR60BHA2 decodes a work-mode report as a breathing rate.

Meanwhile:

- **Cogs already read sensors.** The cog ecosystem (the module explorer) already ships readers (cogs) for LD6002, LD2450, LD1040C, RD-03E, SEN0628 ToF and others. Each is a small signed program that writes JSON lines.
- **A shared format already exists.** `spatial.evidence.v1` (WeftOS ADR-111, first drafted in the spatial workspace ADR-107 §7) is shared by RuView's RF export (ADR-382, `ruview-spatial-evidence`) and by the radar cogs (`--spatial-out`). It already defines `radar_track_point`, `radar_range`, `pose`, `shell_measure`, `human_confirm`, `tof_depth`, `uwb_echo` and `imu_event`.
- **RuView already rejected bespoke ingest paths.** ADR-320 (Sensor HAL) decided against "one bespoke ingest path per modality". `SensorHal` exists in `ruview-hal`, but it lives in-process only, and nothing in production uses it.

The missing piece is an **inbound** contract. Something outside RuView needs a way to say "here is evidence from my sensor" without new Rust in this repository.

## 2. Two contracts, two speeds

| Contract | Carries | Rate | State today | Step |
|---|---|---|---|---|
| **Evidence** (`spatial.evidence.v1`, JSONL) | Radar tracks and ranges, ToF, UWB, IMU events, node poses, room shell, operator labels | Low: up to ~30 lines/s per LD2450 | Export only (ADR-382) | 1, 2 |
| **RF frames** (binary UDP) | Raw CSI from any radio, plus edge packets | 20–1000 frames/s per link | De facto: whatever `csi_collector.c` sends. 9 magics, no single versioned spec | 3 |

They are kept separate on purpose. JSONL is the wrong encoding for raw CSI, and the CSI frame format should not have to change to admit a radar.

### 2a. The long tail: readings that are not spatial

Temperature, humidity, pressure, CO2, VOC and other gases, light, sound level, door and contact state, power draw, and the hundreds of other quantities in the module catalog (several hundred modules and chips, and growing as manufacturer catalogs are ingested) do not fit `spatial.evidence.v1`. Its envelope requires `frame`, `region` and `uncertainty_m` in metres, and a temperature reading has no position uncertainty in metres. Today each cog reports these ad hoc, with the unit encoded in the field name (`temp_c`, `pressure_hpa`, `accel_g`). There is no shared unit or quantity vocabulary.

**Proposal: a third contract, generic readings, built on SenML.**

- **Wire:** SenML (IETF RFC 8428) in its JSON form, one pack per line.
  - Every record carries a name (`n`), a unit (`u`, from the IANA SenML Units registry: `Cel`, `%RH`, `Pa`, `ppm`, `lx`, `dB`, `W`, …), a value (`v`, `vb`, `vs` or `vd`) and a time (`t`).
  - It also has base fields (`bn`, `bt`, `bu`) for compact batches.
  - It is an existing standard with existing producers (LwM2M devices, many gateways), so RuView would not be inventing a schema.
- **RuView profile:** a short, documented subset with mandatory fields. It uses SenML's extension rules; must-understand fields end in `_`:
  - `bn`: the source id, with the same id rule as the evidence envelope.
  - `proof_`: `MEASURED`, `CODE` or `SYNTHETIC`. The trailing underscore means a reader that does not know the field must reject the record, so proof can never be silently dropped.
  - `entity`: optional, the id of the object or space the reading describes (an ADR-306 `Object` or `Space`, for example `space:bedroom` or `object:fridge`). When it is absent, a binding table supplies it (§2b).
  - `sigma`: optional, the 1σ uncertainty in the reading's own unit.
- **Quantity names:** a controlled vocabulary for `n` (for example `temperature`, `humidity`, `co2`, `tvoc`, `pm2_5`, `illuminance`, `sound_level`, `contact`, `power`).
  - Each name is bound to exactly one SenML unit, so `temperature` always uses `Cel`.
  - The vocabulary is a versioned file in the repo, not code. Adding a quantity is a one-line change with a test vector.
  - Unknown names are accepted, stored and counted, but nothing consumes them until they are added to the vocabulary.
- **Same doors, same rules:** the readings ride the same push, pull and file transports, auth, allowlist and rate limits as step 2. The server dispatches on content: a line with a `schema` field is evidence, a JSON array is SenML.

### 2b. Readings describe objects; evidence describes people

The important split is not between sensor types. It is about **what a measurement is about**.

| Kind | Subject | Examples | What RuView does with it |
|---|---|---|---|
| **Perception evidence** | The occupants of a space, and the geometry they move through | CSI, radar tracks, ToF depth, UWB ranges, mmWave vitals, PIR | Fuses it to infer people: presence, count, position, pose, activity, vitals, each with uncertainty |
| **Object readings** | The state of an object or a space | Room air temperature, fridge door open, stove power, CO2 in a room, light level | Updates the state of that entity in the world model. Projections use it to show the view; it can change how data is drawn or interpreted. It is **not** evidence about people by default |

A reading crosses into perception in two ways, and only through an explicit, declared link. Nothing is inferred from a sensor's type alone.

1. **Interaction cue.** Some changes of object state imply that a person acted: a door opened, a light switched on, a stove drew power, CO2 rose in a closed room. Each such cue is declared per quantity and entity with its own likelihood (for example, that a door contact opening means someone is within a metre or two of the door within a few seconds).
   - Cues enter at the validation plane first, as corroborating references (step 3a).
   - They feed fusion only once their likelihood is measured.
   - They never decide presence on their own.
2. **Sensor covariate.** Some readings change how a sensor behaves rather than describing people. Temperature and humidity shift CSI amplitude and phase, and a metal door's state changes multipath. They feed calibration and domain-state checks (ADR-301/302), never the person estimate.

Everything else is object state and stays object state.

**Model changes this needs (ADR-306 amendment):**
- **The gap:** the ontology has `Space`, `Zone` and `Object` entities, but no way to say "this reading is property P of entity E". `Observation` has only `located_in`, with no subject and no value.
- **Change 1, a `Reading` fact.** Add `Reading { subject: EntityRef, property, value, unit, at_unix_ms, evidence_level, provenance }`.
- **Change 2, entity state.** Give `Space` and `Object` a bounded current-state map built from readings.
- **Change 3, a binding table.** It maps a source and quantity to an entity and property, for example `sen-temp-1/temperature → space:bedroom.air_temperature`. It is configuration, not code. A record's `entity` field overrides it.
- **Cue and covariate declarations** live alongside the quantity vocabulary, with a test vector each.
- **HAL:** perception evidence lifts into `HalObservation`. Readings lift into `Reading`, a different type, so fusion cannot mistake one for the other.

**Vitals are about a person, not an object.** Heart rate and breathing measured by a sensor (for example a 60 GHz radar) may travel as SenML. Their subject is then an anonymous track or a space-level "occupant" slot, never an `Object`. They also stay behind the calibrated vitals publication gate (#2147) like any other vitals source: SenML is the wire format, and it grants no authority to publish. A vitals record without a track or space subject is rejected. Until a track binding exists, external vitals are stored and counted but neither published nor fused.

**Privacy.** Object readings can reveal behaviour as clearly as presence does. Stove power, door contacts and light patterns make a daily routine. They fall under the same privacy mode and retention rules as perception data.

### 2c. Where RuView stops

RuView's job is to **fuse every input it can get into one view of a space**: its geometry, the state of its objects, and the people in it, each with uncertainty and evidence level. Inside that boundary it should take everything available. Outside it, it hands off.

| In scope | Out of scope (owner) |
|---|---|
| Ingesting any producer that speaks a contract: frames, evidence, readings | Acquisition, drivers and firmware per device (producers, cogs) |
| The world model: spaces, objects and their state, sensors and their poses | Long-term telemetry history and dashboards (WeftOS, Homecore) |
| People inference: presence, count, position, pose, activity, vitals, plus the gates on publishing them | Automation and actuation, such as turning the heat up (Homecore, ADR-128) |
| Declared cues and covariates linking readings to people and sensors | Rendering: projections and clients read the world model (WeftOS ADR-107 §10) |
| Evidence levels, calibration and qualification of every output | Identifying people beyond anonymous tracks (excluded by privacy policy) |

Republishing the world model, for example over MQTT (#2117) or as `spatial.evidence.v1` export (ADR-382), stays in scope, because that is how projections and automation receive the view.

The work slots in as **step 2b** (parser, vocabulary file, vectors, server dispatch), after step 2 and before 3a, so that 3a can use CO2 and contact sensors as additional references. ADR-384 is extended to cover both contracts, or ADR-387 records the readings contract separately; see §9 Q8.

## 3. Ground truth: what the code has today

All references are on `main` @ `0b15c0eb`.

### Evidence crate (`v2/crates/ruview-spatial-evidence`)
- **Envelope and validation (`wire.rs:87-157`):**
  - The envelope is `EvidenceRecord` with a flattened `RecordBody`, internally tagged by `type`.
  - Ids must match `[A-Za-z0-9._:/-@]{1,128}`.
  - `uncertainty_m` must be in (0, 100].
  - Coordinates must be within ±1000 m.
  - A line is capped at 16 KiB.
- **What it accepts:** `RecordBody` has two variants, `rf_gaussian` and `rf_link_observation`. Any other type fails as `EvidenceError::Parse`; there is no dedicated `UnknownType`. The test `non_rf_types_are_unknown_to_this_emitter` (`tests/evidence.rs:383`) pins that behaviour.
- **Proof tags:** `ProofTag` is `Synthetic | Code | Measured`, and v1 has no CLAIMED.
- **Dependencies:** `ruview-unified` (about 5.8k LOC, plus ndarray and rand). The dependency is only needed for the RF export.
- **Reference validator:** it lives in `weftos-spatial-core/src/evidence/validate.rs`, and its golden lines are the WeftOS ADR-111 vectors (`contracts/sensors/vectors/`). RuView's `tests/evidence.rs:128,155` already copies those lines.

### Sensing server (`v2/crates/wifi-densepose-sensing-server`)
- **Router:** an axum chain at `main.rs:15047-15234`. Auth layers are applied in this order: `AuthState` (15200), privacy filter (15208), `require_bearer` (15222), host allowlist (15231). A route added after 15222 bypasses auth (comment at 15212-15221).
- **Auth scopes:** non-GET routes need `sensing:admin` (`bearer_auth.rs:455,481`). The vendor `/events` route is the one existing exception (`:463`).
- **Closest ingest precedent:** `ingest_vendor_events(State, Path, Bytes)` (`main.rs:6280`, routed at 15085).
- **Limits:** there is no `DefaultBodyLimit`, so axum's 2 MB default applies, and there is no HTTP rate limiting.
- **Shared state:** `AppStateInner` (`main.rs:1975`), held in a single `Arc<RwLock<…>>`.
- **Status endpoints:** `/api/v1/status` is served by `health_ready` (`main.rs:7619`). UDP counters appear only on `/health` (`processing_health`, 11418).
- **CLI:** the CLI struct is at `main.rs:102-337`. Flags are kebab-case, env vars use `RUVIEW_*`, and booleans go through `cli::parse_env_bool`. `--data-dir` and its helpers are at 185, 8135 and 8143.
- **Node positions:** set with `--node-positions` (`main.rs:300`) and parsed at 14718-14750.
- **Missing dependencies:** the server depends on none of `ruview-hal`, `ruview-fusion`, `ruview-spatial-evidence`, `ruview-unified` or `ruview-groundtruth`.
- **Calibration:**
  - `calibrated_presence_evidence` (`main.rs:2767`) uses the eigenvalue field model only.
  - The only operator-reference hook is `POST /api/v1/config/ground-truth`, which takes `{count}` (`main.rs:15828`).
- **Ground-truth labeller (`v2/crates/ruview-groundtruth`):** it runs offline and takes `RadarWindow` (`radar.rs:85`: presence, target_count, distance, targets with x/y), then `label_window`, `RadarLabels` and `auto_label`. The header says it "stays on the validation plane: nothing here feeds an estimator."
- **Integration tests:** they spawn the binary on free ports (`tests/multi_node_test.rs:281-291`) or call the router in-process with `oneshot` (`tests/rufield_surface_test.rs`).

### HAL and fusion
- **`SensorHal`:** defined as `{ type Raw; describe(); normalize(raw, ctx) -> HalObservation }`, with infallible normalisation (`ruview-hal/src/adapter.rs:47-60`).
- **Modalities:** `Csi, Ieee80211bf, Ble, Uwb, Mmwave, Acoustic, Camera, Lidar, Imu, Custom(String)`.
- **Adapters:** only synthetic reference adapters exist.
- **Fusion:** `ruview-fusion::FusionEngine::fuse(&[PresenceObservation])` consumes `HalObservation`. It is not used by the server.
- **Evidence levels:** the ontology defines `EvidenceLevel` L0-L5, and `Observation.at_unix_ms` is in ms while the wire uses ns.

### Cogs (WeftOS cog repo)
- **`ld2450-radar`:**
  - Spatial output is behind the `spatial-evidence` cargo feature, which is off by default.
  - Its spatial flags are `--radar-pose x,y,z,yaw,pitch[,x_sign]`, `--spatial-region` and `--spatial-source-id`.
  - `--spatial-out FILE` appends lines to a file.
  - `--spatial-out export` serves `GET /spatial?seconds=N` (a 30 s ring, N ≤ 30) on `--api-bind`, default `127.0.0.1:8052`.
  - It writes one line per target per frame at about 10 Hz. Track ids are radar slots 1-3, z is 0, and there is no velocity.
  - `t_ns` is the host parse time.
  - Proof is `MEASURED`, or `SYNTHETIC` with `--simulate`.
  - It writes nothing when there are no targets.
- **`ld6002-radar`:** file output only.
- **`ld1040c-motion`:** has no spatial output.
- **Portability:** `serialport` 4 without libudev is cross-platform, and the cogs are plain Rust. An x86_64 Linux host build of one cog exists, which shows the cogs build for x86. The cog repo CI cross-builds only armv7/aarch64, and `ld2450-radar` currently ships only as ARM builds. The 256000-baud path is noted as termios2 (Linux); macOS needs a check.
- **Bridge cog `--relay-to`:** it relays 8-float vectors only, not evidence lines.

## 4. Step 1: inbound evidence parsing (library only)

**Goal:** RuView can read and validate every `spatial.evidence.v1` type it needs, using the same rules as the reference validator. No server changes.

### Changes
1. **Feature-gate the RF export in `ruview-spatial-evidence`.**
   - The `rf-export` feature is on by default and carries the `ruview-unified` dependency plus `gaussian.rs` and `link.rs`.
   - The new `ingest` code depends only on serde, serde_json and thiserror. The server can then use `default-features = false` without pulling in ndarray.
2. **Add the inbound bodies to `RecordBody`:** `radar_track_point` (with the optional `sensor` block), `radar_range`, `pose`, `shell_measure`, `human_confirm`, `tof_depth`, `uwb_echo` and `imu_event`. Each body is validated by the rules in WeftOS ADR-111 §1-§2 (table in Appendix A).
3. **Add `EvidenceError::UnknownType(String)`** so the server can count unknown types separately from malformed lines.
4. **Make the emitter safe after the change:** `to_line` keeps refusing anything other than the two RF types unless the `ingest` feature is on. ADR-382's emitter rule ("does not mirror the other v1 types") is amended in the same change.
5. **Add test vectors** in `v2/crates/ruview-spatial-evidence/tests/vectors/`:
   - `valid/*.jsonl`: each ADR-111 golden vector, plus edge values.
   - `invalid/*.jsonl`: one file per rejection rule, with the expected error in a sidecar `.expect`.
   - The vectors are generated from, and diffed against, the ADR-111 text and vectors, so the two validators cannot drift silently.
6. **Add a check tool:** `cargo run -p ruview-spatial-evidence --example evidence_check -- FILE` prints accepted and rejected counts per type and reason, and exits non-zero on any rejection. It becomes a CLI subcommand later if wanted.
7. **Update the tests:** flip `non_rf_types_are_unknown_to_this_emitter` to cover both sides: the emitter still refuses, and the reader accepts.

### Gates
- `cargo test -p ruview-spatial-evidence` with `--no-default-features` and with default features.
- `cargo clippy -p ruview-spatial-evidence --all-features -- -D warnings`.
- A cross-check that every vector passes or fails identically under `weftos-spatial-core`'s `parse_line`. This runs locally only, because that crate is not a RuView dependency.

### Exit criteria
- All ten v1 types round-trip.
- Every rule in Appendix A has a rejecting vector.
- RuView and the reference validator agree on every vector.

**Size:** about 600-900 lines including tests. **Risk:** low, since it is library-only.

## 5. Step 2: server ingest (additive endpoints)

**Goal:** the sensing server accepts evidence by push, pull or file, keeps a bounded recent window, and exposes counters. Nothing consumes the data yet except the status endpoints and recordings.

### Transports (all share one validator and one per-source limiter)

| Transport | Shape | Why |
|---|---|---|
| **Push** | `POST /api/v1/evidence`, JSONL body ≤ 1 MiB (route-level `DefaultBodyLimit`) | Any producer that can POST |
| **Pull** | `--evidence-pull URL[,URL…]`: poll a producer's `GET /spatial?seconds=N` every N/2 s and dedupe by `provenance.receipt` | Works with `ld2450-radar --spatial-out export` **with no cog changes** |
| **File** | `--evidence-file PATH` (replay once) and `--evidence-follow PATH` (tail) | CI, recorded sessions, and producers that only write files (`ld6002-radar`) |

A WebSocket transport (`/ws/evidence`) is deferred to after step 3.

### Trust and limits
- **Mounting:** the ingest routes are mounted only when `RUVIEW_API_TOKEN` is set. They are added before the `require_bearer` layer (`main.rs:15222`), with a test that the route returns 401 without a token.
- **Scope:** push requires `sensing:admin`, the existing default for non-GET routes. A narrower `evidence:ingest` scope is an open question (§9).
- **Source allowlist:** `--evidence-sources ld2450-1,ld6002-1` (env `RUVIEW_EVIDENCE_SOURCES`). An empty list means everything is rejected. Unlisted `source_id`s are rejected and counted.
- **Per-source token bucket:** 100 lines/s sustained with a burst of 500, configurable. Excess lines are dropped and counted, never queued without bound.
- **Clock model:** every record is stored with two times: the producer's `t_ns`, exactly as sent, and the server's receive time.
  - **Freshness:** the bounds apply to the receive time (5 s future skew, `--evidence-max-age` default 30 s past). The producer's time can therefore be corrected later without the record being rejected as late.
  - **Corrections:** producers may publish clock-correction rows of anchor, offset and rate, the drift-row model from the WeftOS sound-and-radio note. RuView keeps the latest fit per source and computes shared time as `t_shared = t_local − (offset + rate × (t_local − t_anchor))`. When a new row closes an interval, the fit is applied back across buffered records from that source. Records are never rewritten on disk; the correction is a separate stored fact.
  - **Fallback:** until a correction record type exists in `spatial.evidence.v1` (§9 Q9), RuView uses the producer time as is and reports skew per source.
- **Room bounds:** once a `shell_measure` is known for a region, positions outside it plus `uncertainty_m` are rejected.
- **Region registry:** a `region` id must be registered before its records are used. Registration happens through `--evidence-regions` or the first `shell_measure` for that id, and resolves the region to a room shell and frame. Records for an unregistered region are validated, stored and counted, but no consumer uses them. A valid label like `test-room` therefore cannot silently enter the world model.
- **Privacy:** under `--privacy-mode` (#2125), track positions and object readings are not exposed on REST or WebSocket and are not recorded. Only counts and presence are. This matches ADR-384 §7; readings such as stove power or door contacts reveal routines as clearly as presence does (§2b).

### State and exposure
- **`EvidenceStore` in `AppStateInner`:**
  - Per source: a ring buffer of 30 s or 4096 records, whichever is smaller.
  - Latest `pose` per node and latest `shell_measure` per region.
  - Counters: accepted, rejected by reason, rate-limited, dedup hits, and clock skew per source.
- **`GET /api/v1/evidence/status`:** shows the counters plus each source's last-seen time and its rate.
- **`GET /api/v1/evidence/latest?seconds=N`:** returns recent records (privacy rules apply).
- **`/api/v1/status`:** gains a short `evidence` summary block.
- **Recordings:** while a recording is active, accepted lines are written to `<data-dir>/recordings/<session>.evidence.jsonl`, alongside the CSI recording. This is what step 3's consumers read.

### Gates
- `cargo test -p wifi-densepose-sensing-server` with default features and with `--features mqtt`.
- New integration tests in `tests/evidence_ingest.rs` using the spawned-binary pattern on free ports. They check:
  - push accepted and rejected counts;
  - 401 without a token;
  - the 413 body limit;
  - the rate limit;
  - pull from a stub `/spatial` server, with dedupe;
  - file replay determinism, with identical counts across 3 runs;
  - privacy-mode suppression.
- Clippy on the changed lines, `docker build -f docker/Dockerfile.rust .`, and the security review checklist from CLAUDE.md for the new write endpoint.

### Exit criteria
- Replaying a recorded LD2450 JSONL gives the same counts on every run.
- All the rejection paths above are counted.
- With no evidence flags set, behaviour is byte-for-byte the same as before: the existing test suites pass unchanged.

**Size:** about 900-1300 lines including tests. **Risk:** medium, because it is a new write endpoint, mitigated by the controls above.

## 6. Step 3: first consumers, and the frame-format record

These are two independent tracks, and either can go first.

### 3a. Evidence consumers (ADR-384 phases 3-4)
1. **Ground-truth adapter.** Add `ruview_groundtruth::radar::windows_from_evidence(&[EvidenceRecord], window_ms)`, which turns `radar_track_point` and `radar_range` records into `RadarWindow`s.
   - It sets presence, target count, distance to the sensor and targets x/y.
   - It sets `source.verified` only when proof is `MEASURED`.
   - `auto_label` then works directly on a recording plus its `.evidence.jsonl`.
   - It stays offline and on the validation plane, as the crate already requires.
2. **Calibration reference diagnostics.** This is read-only and never a gate.
   - During and after calibration, `GET /api/v1/calibration/status` reports agreement between `calibrated_presence_evidence` and the evidence reference: radar track presence and count, plus any `human_confirm` boxes.
   - It reports the agreement fraction over a time window, with sample counts.
   - This provides the occupied (walking and seated) controls that the #2147 review found missing (#1952). The detector's decisions are not affected.
3. **HAL adapter (groundwork).** `EvidenceHal: SensorHal<Raw = EvidenceRecord>` in `ruview-hal`.
   - Modality mapping: radar types to `Mmwave`, `uwb_echo` to `Uwb`, `imu_event` to `Imu`, `tof_depth` to `Custom("tof")`.
   - Proof mapping: `SYNTHETIC` to L0 with the synthetic calibration handle, `CODE` to L1, `MEASURED` to L2. Higher levels come only from fusion or calibration, never from the wire.
   - `t_ns` becomes ms.
   - It is tested in-crate only, because the server does not run `ruview-fusion` yet. That is stated plainly in the ADR.
4. **Hardware validation (MEASURED).**
   - **Bench:** an LD2450 on a USB-UART adapter, with `ld2450-radar --spatial-out export` running on the Mac (or a Pi) and the server pulling with `--evidence-pull`.
   - **Room:** the 12×12 ft room (3.66 m), measured with tape. That gives one `shell_measure`, a `pose` for each node and the radar, and the router outside the room at a measured spot (separate plan).
   - **Protocol:**
     - Empty for 10 min, with no pets; record that fact.
     - Walking for 2 min.
     - Seated for 2 min.
     - Operator `human_confirm` boxes for each phase.
   - **Results:**
     - Radar track presence against the operator labels.
     - Calibrated presence against both.
     - Every result reported with sample counts, the capture command, and evidence tags.
     - The validation record goes in `docs/validation/`.

### 3b. RF frame contract record (ADR-385), documentation and tests only
- **Write ADR-385, "ESP32 UDP frame contract v1".** It describes exactly what the firmware sends today, with no format changes:
  - **The 9 magics:**

    | Magic | Frame |
    |---|---|
    | `0xC5110001` | CSI (ADR-018) |
    | `0xC5110002` | vitals |
    | `0xC5110003` | features |
    | `0xC5110004` | fused vitals |
    | `0xC5110005` | compressed CSI |
    | `0xC5110006` | feature state |
    | `0xC5110007` | WASM output |
    | `0xC5118100` | mesh |
    | `0xC511A110` | ADR-110 sync |

  - Their byte layouts, endianness, sizes and version fields, citing the firmware definitions (`csi_collector.c:154-185`, `edge_processing.h`, `rv_feature_state.h`, `wasm_runtime.h`, `rv_mesh.h`).
  - It points to ADR-267 for MediaTek MTC1 instead of duplicating it.
- **Resolve a naming conflict.** `wifi-densepose-hardware/src/esp32_parser.rs:59-60` labels `0xC5110007` as ADR-095 temporal-classification, while the firmware and server treat it as ADR-040 WASM output (#928). The ADR records which is current, and the parser comment is fixed. This is a comment change only.
- **Document the timing that exists today.**
  - The CSI frame (`0xC5110001`) has no timestamp field. ADR-110 §A0.11-A0.12 added a 32-byte sync packet (`0xC511A110`: node id, flags, node-local `local_us`, mesh-aligned `epoch_us`, high-water CSI sequence). It is sent about every 20 CSI frames.
  - The server already parses it (`main.rs:12690`) and keeps a per-node offset (`apply_sync_packet`, `main.rs:1445`), which `multistatic_bridge.rs` uses.
  - ADR-385 records how a CSI frame is joined to sync packets by `(node_id, sequence)`. The current model is **offset only**, with no rate term and no back-correction.
  - ADR-110 measured about 30 µs/min of drift between two boards, which is exactly the ramp an offset-only model leaves between sync packets. Adding a rate term (anchor, offset, rate), with fits applied back over the interval, is recorded as the next versioned change of the sync packet. It is not changed in this step.
- **Add golden vectors** in `v2/crates/wifi-densepose-hardware/tests/vectors/`: the real C6 frames already in `tests/adr110_live_frames.rs`, plus sanitised S3 and C5 captures (MACs zeroed, no person data). A test parses each one with the Rust parsers.
- **Add a check tool:** `frame_check` reads a pcap or raw datagram dump and reports counts per magic and parse errors. With it, any CSI producer (another firmware, the MediaTek bridge, a cog) can show it conforms.

### Gates
- 3a: crate tests for `ruview-groundtruth` and `ruview-hal`, plus server tests for the calibration-status fields, and the hardware run itself for item 4.
- 3b: `cargo test -p wifi-densepose-hardware`. No firmware change. `firmware-ci` stays as it is.

## 7. Step 4: later, case by case (not planned in detail)

Once steps 1-3 are proven, each firmware extra gets its own short ADR, which decides whether the extra stays in firmware or moves to a producer.
- **Candidates:** the UART1 radar path (now duplicated by cogs), edge vitals, 802.15.4 time-sync (#2162) and the mmWave fusion packet (`0xC5110004`).
- **Rule:** a path is retired only after its replacement has run side by side and matched on recorded data.
- **ESP32 CSI capture:** it is not a candidate. It remains the reference CSI producer, governed by ADR-385.

## 7a. Producer packaging: RVF (proposal, after step 3)

Steps 1-3 fix **what** a producer sends. This section covers **how a producer is packaged and run**, so that one artifact works on a Mac, a Pi, a server, and an ESP32 node.

### What exists
- **RVF** (ruvector ADR-029/ADR-030) is a segmented single-file container with 64-byte segment headers. Relevant segments:
  - `MANIFEST`, plus `CRYPTO` signatures (ML-DSA-65) and a `WITNESS` chain.
  - Three execution tiers:

    | Tier | Segment | Status |
    |---|---|---|
    | 1 | `WASM_SEG` (portable, ~KB) | exists |
    | 2 | `EBPF_SEG` | ADR-030, Proposed |
    | 3 | `KERNEL_SEG`: a unikernel for x86_64/aarch64/riscv64 under Firecracker/QEMU/Uhyve | ADR-030, Proposed |

  - ADR-030 is explicit that WASM has "no direct hardware access" without a host runtime.
- **RuView already speaks RVF:**
  - `sensing-server/src/rvf_container.rs` reads and writes the segment format without the `rvf-wire` crate.
  - ADR-003 and ADR-009 package CSI fingerprints and edge models as RVF.
- **The ESP32 firmware already runs WASM (ADR-040, Accepted):**
  - wasm3 is in `main/wasm_runtime.c`. Modules link against a host-import namespace, `"csi"` (`csi_get_phase`, `csi_emit_event`, …).
  - Modules are hot-loaded through `POST /wasm/upload` (max 128 KB).
  - Output leaves the node as `0xC5110007` packets.

### Proposal: a producer is a signed RVF with a WASM decoder
1. **Split each producer into I/O and a decoder.** The decoder is pure: protocol bytes in, `spatial.evidence.v1` lines out, with no OS calls. It compiles to `wasm32-unknown-unknown`. I/O (opening the serial port, timing) belongs to the host.
2. **Package it as an RVF:**
   - `MANIFEST` holds the producer id, the sensor, and the host capabilities it needs (for example `uart: 256000 8N1`, `rate_hz: 10`).
   - `WASM_SEG` holds the decoder.
   - `CRYPTO` holds the signature.
   - Native per-architecture binaries are optional extra segments for producers that need full OS access.
3. **Define one small host ABI, `"io"` + `"evidence"`.** It is implemented by every host:
   - `io_read(buf, len) -> n` reads bytes from the bound device.
   - `io_write(buf, len)` sends commands to the sensor.
   - `evidence_emit(ptr, len)` hands over one JSONL line.
   - `clock_ns()` returns the time.
4. **Hosts:**
   - **Any computer:** a small runner (wasmtime) opens the serial port named in the manifest, runs the decoder, and pushes to ingest. One `.rvf` covers macOS, Linux and Windows on any CPU. This removes the "ARM-only build" problem (§9 Q6) for decoders.
   - **ESP32 node:** the firmware's wasm3 runtime gets the same `"io"` and `"evidence"` imports, bound to UART1. Emitted lines travel to the server, either in the existing `0xC5110007` WASM output packet or in a new evidence magic recorded in ADR-385.
   - **Result:** the firmware becomes a **generic sensor host**. A new radar on a node's UART means uploading a signed decoder, not writing a firmware driver. That is the route to retiring the UART1 radar driver in step 4.

### Constraints and open points
- **Hardware access stays in the host, by design.** A KERNEL_SEG unikernel does not help with a USB-UART on a laptop, so tier 3 is out of scope here.
- **wasm3 on the ESP32 is an interpreter with a 128 KB module cap.** That suits ~10 Hz radar decoding but not CSI-rate work. The decoder size and per-frame cost must be measured on hardware before any claim is made.
- **Existing cogs are native Rust using `serialport`.** Each one would need the decoder/I-O split on the cog side; that is not RuView work. A cog can keep shipping native binaries next to the WASM decoder in the same RVF.
- **Signature verification on the node already exists.** `POST /wasm/upload` (`main/wasm_upload.c`) accepts an RVF container and, by default, verifies its signature against a public key provisioned into NVS (`provision.py --wasm-pubkey`, ADR-050). Raw unsigned `.wasm` is accepted only when `CONFIG_WASM_SKIP_SIGNATURE` is set for lab builds. What still needs a decision is key management for decoders published by third parties: who signs them, and how a node trusts more than one publisher key. That ties into ADR-305 identity.
- **The RVF segment type for a producer manifest** should be agreed with the RVF format owners rather than invented here.

This needs its own ADR (ADR-386, proposed) after step 3. It does not change steps 1-3, because a producer packaged this way emits the same evidence lines.

## 8. PR slicing and order

| PR | Contents | Depends on | Reviewable alone |
|---|---|---|---|
| A | ADR-384 + step 1 (crate feature split, inbound types, vectors, check tool), ADR-382 amendment | none | yes |
| B | Step 2 server ingest | A | yes |
| B2 | Readings: SenML profile, quantity vocabulary, binding table, ADR-306 `Reading` + entity state, vectors, server dispatch. Cues and covariates are declared but not consumed. | B | yes |
| C | 3a items 1-3 (ground-truth adapter, calibration diagnostics, HAL adapter) | B | yes |
| D | 3a item 4: validation record (docs only) | C + hardware | yes |
| E | 3b: ADR-385, vectors, frame-check tool, parser comment fix | none | yes |

A and E can be reviewed in parallel. Each PR carries its own test evidence and claims nothing beyond it.

## 9. Open questions for the reviewer

1. **Crate shape.** Should inbound types grow inside `ruview-spatial-evidence` (feature-split, ADR-382 amended), or go in a separate decoder crate? The recommendation is the feature split, because it keeps one validator. Note that an unrelated `ruview-evidence` crate (the ADR-304 evidence engine) already exists, and the README should spell out the difference.
2. **Push auth scope.** Reuse `sensing:admin`, or add `evidence:ingest` so a radar host's token cannot change server config? The recommendation is to add the narrow scope in PR B if `ruview-auth` supports custom scopes, and otherwise use admin and follow up later.
3. **Pull by default.** Pull needs no producer changes, but it makes RuView open outbound connections to configured URLs. Those URLs come only from CLI or env, never from evidence content. Is that acceptable to ruvnet's threat model?
4. **Clock.** *Proposed in §5:* store both the producer time and the receive time, apply freshness bounds to the receive time, and accept published offset and rate corrections. Is applying corrections back over buffered records acceptable to reviewers, given that it changes the time of data a client may already have read? The proposal is that `/evidence/latest` returns both the raw and the corrected time.
5. **CLAIMED.** v1's proof tags are SYNTHETIC, CODE and MEASURED, with no CLAIMED. RuView's evidence policy uses CLAIMED. Should this be mapped (to CODE on export, never to MEASURED), or proposed upstream as an addition to v1?
6. **Portable producer builds.** The cog repo CI builds armv7/aarch64 only. x86 builds already work for at least one cog, but `ld2450-radar` ships ARM-only. A macOS or x86 build of it is needed for the Mac bench in step 3a. Is that tracked on the cog side? The 256000-baud path also needs a macOS check.
7. **Upstream framing.** Should ADR-384 be proposed with ADR-385, or should 384 go first so the direction is seen working before the frame record?
8. **Readings contract.** *Partly settled (2026-10-05):* WeftOS agrees with the SenML (RFC 8428) profile as described in §2a, so cogs and RuView converge on it. The ADR-384 draft now covers both inbound contracts; a reviewer may still ask for the readings part to be split into ADR-387. *Answered 2026-10-07:* the shared quantity vocabulary is published in WeftOS as `contracts/sensors/quantities.v1.json` (ADR-111), which both RuView and the cog catalog consume.
9. **For WeftOS: new evidence record types.** The sound-and-radio and snapshot-sounding notes (`docs/design/`, 2026-10-06) need three record types that `spatial.evidence.v1` does not have:
   - an **acoustic range**, which is not `tof_depth` (optical) and not `uwb_echo`;
   - a **cooperative radio delay** between two scheduled nodes, with its direction and whether it is one-way or round-trip;
   - a **clock correction** row (node, anchor, offset, rate, residual).
   
   WeftOS owns the schema. *Answered 2026-10-07:* WeftOS ADR-111 defines all three (`acoustic_range`, `radio_delay`, `clock_correction`); RuView adds them in a follow-up change.
10. **Later consumer: a shell-aware empty-room baseline.** Once `shell_measure` and `pose` are known, the empty room's direct and first-bounce paths can be predicted from the outline. The existing Fresnel baseline (`wifi-densepose-signal/src/fresnel.rs`, `fresnel_confidence`) then becomes a prediction, and the residual on named paths is the person. This would be the first consumer that uses evidence geometry for people inference rather than validation. It needs its own ADR, and delay resolution stays bandwidth-limited (about 15 m at 20 MHz Wi-Fi). Is this the right next consumer after step 3a, and should WeftOS's spatial engine or RuView compute the predicted paths?
## 10. Risks

| Risk | Mitigation |
|---|---|
| Two validators (RuView, weftos-spatial-core) drift | Shared golden vectors built from the ADR-111 text and vectors; cross-check in step 1 |
| New write endpoint abused | Token, allowlist, body limit, per-source rate limit, bounds, typed rejections, counters |
| Person positions are personal data | Privacy mode suppresses positions; bounded retention; no export by default |
| Producer clock skew corrupts alignment | Bounds on `t_ns`, skew reported per source, dual timestamps (question 4) |
| Scope creep into fusion or estimators | Consumers stay on the validation plane (labeller, diagnostics); the HAL adapter is library-only until a fusion ADR |
| Reviewer load upstream | Six small PRs (five code PRs plus a docs-only validation record), each standalone, each with its own evidence |
| Cog not buildable on the bench host | Pull also works from a Pi; the file transport works from any host |

## Appendix A: inbound body rules (from WeftOS ADR-111, origin spatial ADR-107 §7, §7.1, `weftos-spatial-core/src/evidence/validate.rs:126-210`)

| Type | Required | Optional | Rules |
|---|---|---|---|
| `radar_track_point` | `track` u32, `position` [3] | `velocity` [3], `sensor` {position[3], yaw_deg, pitch_deg=0, elev_half_deg, az_half_deg} | velocity components within ±50; yaw ±360; pitch ±90; elev_half 0.1-89; az_half 0.1-180 |
| `radar_range` | `position`, `yaw_deg`, `fov_deg` [h,v], `range_m` | `pitch_deg`, `targets` | fov in (0, 180); range in (0, 1000]; targets 0-64 |
| `pose` | `node_id`, `position` | `yaw_deg` | yaw ±360 |
| `shell_measure` | `min`, `max` | none | min < max on every axis |
| `human_confirm` | `min`, `max`, `label`, `occupied` | none | ordered box; label 1-64 characters, no control characters |
| `tof_depth` | `position`, `yaw_deg`, `pitch_deg`, `fov_deg`, `grid` [c,r], `range_mm`, `valid` | `roll_deg` | grid sides 1-16; both arrays exactly c·r long; ranges ≤ 50 000; roll ±180 |
| `uwb_echo` | `anchor`, `range_m` | `direction` | range in (0, 1000]; direction non-zero |
| `imu_event` | `node_id`, `event`, `magnitude` | none | magnitude 0-1e4 |

Envelope rules:
- `schema == "spatial.evidence.v1"`, checked before typed parsing.
- `frame == "room_enu"`: SW floor corner, x east, y north, z up, metres.
- Ids follow the pattern above.
- `uncertainty_m` in (0, 100], 1σ.
- `t_ns` u64, Unix ns.
- Coordinates within ±1000 m.
- Lines ≤ 16 KiB.
- Unknown extra fields are accepted.
- Yaw is counter-clockwise from +x.
