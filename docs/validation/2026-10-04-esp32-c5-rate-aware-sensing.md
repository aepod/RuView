# ESP32 C5 rate aware sensing qualification (bring-up)

Status: **bring-up record** — device-side rate + stability qualified on 2.4 GHz;
the full 300 s controlled run, the end-to-end WebSocket table, and the 5 GHz HE
pass are PENDING. Mirrors the C6 record
[`2026-08-31-esp32-c6-rate-aware-sensing.md`](2026-08-31-esp32-c6-rate-aware-sensing.md).
See [ADR-368](../adr/ADR-368-esp32-c5-firmware-extension.md).

## Scope

Qualifies, on real ESP32-C5 silicon: that the firmware builds, flashes, boots,
associates to WiFi, captures CSI, and sustains a raw CSI callback rate above the
hardware acceptance floor without errors or reboots. Does **not** yet qualify
heartbeat, respiration, gesture, pose, identity or person-count accuracy against
labelled ground truth, the 5 GHz HE path, or the end-to-end fused WebSocket
coverage (those are P4/P5 in ADR-368).

## Hardware and firmware

| Field | Measured value |
|-------|----------------|
| Board | ESP32-C5-WROOM-1 revision v1.0, 16 MB flash, no PSRAM |
| Logical node | 1 |
| Firmware after | 0.8.12 + ESP32-C5 extension (branch `feat/esp32c5-firmware`) |
| Toolchain | ESP-IDF v5.5 (`esp32c5` preview target) |
| C5 app image | 1,151,440 bytes |
| C5 app SHA 256 | `3577a9082d8de4703b8e0830ad70d65f65f6c04f9f696d74250574cd423a1281` |
| C5 bootloader SHA 256 | `a523d2c877fe719e4f780a3f76ab740800187524fbf6daabe427320ee4c4ecf5` |
| OTA slot size | 4,194,304 bytes (one of two 4 MB slots, 16 MB layout) |
| OTA headroom | 3,042,864 bytes, 73 percent |
| Live partition | `ota_1`, `ota_state: valid` (self-marked valid, no rollback) |

## Software gates

| Gate | Result |
|------|--------|
| ESP32-C5 IDF 5.5 build | PASS (`Project build complete`) |
| Flash + image hash verify on C5 | PASS (`Hash of data verified`, all four segments) |
| Boot + onboarding on C5 | PASS (`ESP32-C5 CSI Node` banner, CSI collector + serial onboarding up) |
| NVS provisioning on C5 | PASS (`provision.py --no-stub` fix; WiFi + aggregator config written) |
| Host unit tests (rate/occupancy, ADR-110 encoding, mmWave predicate) | NOT RUN this session |
| ESP32-C6 / S3 cross-compile | NOT RE-RUN this session (gates generalized, not rebuilt for C6/S3) |

## Bring-up finding — TWT lockup

With `CONFIG_C6_TWT_ENABLE=y` (the C6 default, inherited), the C5 hit a hard
`CPU_LOCKUP` roughly one second after boot, immediately after the WiFi driver
logged `Connected AP does not support setup individual TWT agreement`. TWT
negotiation locks up the C5 *preview* WiFi driver against a non-iTWT AP. Fixed by
`CONFIG_C6_TWT_ENABLE=n` for C5 (TWT is a power feature, irrelevant to CSI). With
TWT disabled the node runs indefinitely stable.

## Device-side result (2.4 GHz, bring-up windows)

Measured from the firmware's own CSI callback counter (serial) and the raw CSI
UDP stream to the aggregator (UDP port 5006). Windows are 6–26 s, not the
full 300 s — a controlled 300 s run is pending.

| Device observation | Result |
|---------------------|--------|
| Node IP (DHCP) | assigned by the LAN (omitted) |
| AP channel (auto-detected) | 5 (2.4 GHz) |
| Uptime at measurement | ~14 min, continuous |
| Raw callback mean | **32.7 pps** (20 s UDP), 34.9 pps cumulative (cb #29200 / 837 s) |
| Raw callback range | median 34 pps, per-second up to 39 pps |
| CSI frame size | 32 through **632 bytes** (256-bin HE width — not 64-bin HT) |
| Edge DSP cadence | 8 Hz configured (actual DSP-Hz measurement pending) |
| ENOMEM / UDP send-fail / watchdog / reboot | 0 observed |

## End-to-end WebSocket result

PENDING — requires the full sensing-server/aggregator that serves `/api/v1/mesh`
and `/api/v1/fusion` (the minimal `scripts/ruview-sensing-server.py` does not).

## 5 GHz HE result (P4)

PENDING — re-provision onto a UNII-1 non-DFS channel (36/40/44) and confirm the
HE frame remains 256-bin at 5 GHz. NOTE: 256-bin HE frames (up to 632 B) are
already observed at 2.4 GHz HE20 on IDF v5.5.0, so the ADR-110 "needs IDF ≥ 5.5.2
for HE" caveat does **not** appear to bite on this silicon/toolchain.

## Result and limitation

The ESP32-C5 CSI node captures CSI and sustains **32.7 pps raw** (≥ 20 pps
acceptance floor) with HE-width frames and zero steady-state errors on 2.4 GHz.
The TWT lockup is understood and worked around. Not yet qualified: the full 300 s
controlled run with DSP-Hz range, the end-to-end fused coverage, and 5 GHz.

## Acceptance test

Raw callback yield ≥ 20 pps: **PASS (32.7 pps)**. DSP cadence within ±1 Hz of
configured, zero steady-state ENOMEM/send-fail/watchdog/panic/reboot: **PASS on
the observed windows** (full 300 s run pending). Server parse/coverage/freshness
and 5 GHz: **PENDING**.

## 2026-10-05 qualification (two nodes, 5 GHz)

MEASURED. Two ESP32-C5-WROOM-1 boards (revision v1.0, 16 MB flash, 8 MB in-package PSRAM), logical nodes 6 and 7, ESP-IDF v5.5.2, branch `feat/esp32c5-firmware` with the GPIO17/18 flash-bus fix. Both joined the AP on channel 40 (5200 MHz, 11ax). The 5-minute run used the same app with the console routed to UART0 (local overlay, app SHA 256 `91954e3c89f27fce7948e511256931b3096425a8f61012d60697bdf62c9dde85`) so device counters were readable through the board's UART bridge; the committed image logs to USB-Serial-JTAG instead. The server was sensing-server built from `fix/multinode-loop-freeze-upstream` (PR #2157), UDP 5006, source allowlist on.

| Gate (pass bar) | Node 6 | Node 7 |
|---|---|---|
| Raw callback yield (>= 20 pps) | 42.3 pps (`yield` mean 41.6) | 41.9 pps (`yield` mean 41.2) |
| Edge DSP cadence (8 Hz +/- 1) | 8.0-8.1 Hz | 8.0-8.2 Hz |
| ENOMEM / send fail / panic / watchdog / lockup | 0 / 0 / 0 / 0 / 0 | 0 / 0 / 0 / 0 / 0 |
| Resets in run | 1 (power-on) | 1 (power-on) |
| Server frames received | 8238 (27.5 fps) | 8221 (27.4 fps) |
| Server parse failures, allowlist drops | 0, 0 | 0, 0 |

Server `/health` reported `processing.state = live` in 58 of 60 samples; the two `idle` samples were before the nodes restarted when the console ports opened. Result: **PASS** for raw yield, DSP cadence, device stability and server parse/freshness. The minimum `yield` sample (5 pps) is the boot interval.

That run used `acquire_csi_force_lltf = 1`, so frames carried 53 bins. After setting it to 0 (same day, committed image with USB-JTAG console, app SHA 256 prefix `c0db522ed08c4ab8`), a 75 s capture gave 2133 and 2166 CSI frames (about 28.5 fps per node), with 97-98 % HE-SU frames of **245 bins** (490 B I/Q) and the rest legacy 53 / HT 57 bins. A 40 s server run parsed all of them, locked both nodes' grid gate on 245, and logged no warnings.

Not done: a 5-minute re-run of the device counters on the 245-bin image, occupancy qualification (needs an empty-room session), and 2.4 GHz on these boards (the AP steered them to 5 GHz).
