# Snapshot sounding between our nodes

**Status:** note for the RuView session. Not an ADR. No evidence type is added here.
**Date:** 2026-10-06
**From:** the WeftOS / cog session, after reading SnapPnt and asking whether its capture can range between nodes we schedule.

## The technique

[SnapPnt](https://h-shiono.github.io/snappnt/) (repo [h-shiono/snappnt](https://github.com/h-shiono/snappnt), BSD-2-Clause) is a snapshot GNSS receiver. The radio freezes one block of I/Q. Correlation runs later, on a host, against a local copy of the spreading code. A continuous tracker never starts.

The limit it is written for is the ESP32 debug path documented by [ESP-SDR](https://espargos.net/espsdr/). Some ESP32 chips can dump raw I/Q from the Wi-Fi radio into an internal memory bank. At 80 MSa/s one 64 KiB bank is about 0.2 ms. A NavIC S-band code period is 1 ms, so the bank holds about one fifth of a code. The chip cannot move a continuous stream to a host. Published ESP-SDR firmware implements reception only. ESPARGOS has said arbitrary transmission is possible on their array and has not shipped a transmitter.

Acquisition, in `rx/acquisition.py` and the [architecture note](https://h-shiono.github.io/snappnt/design/architecture/):

- Correlate the burst against a replica that is one full code period plus the length of the snapshot. The lag of the peak is the code phase at the first sample. Every lag is one FFT correlation.
- Repeat that correlation on a grid of carrier offsets. The default step is `1/(2T)`, where `T` is the coherent integration time. The default search is ±30 kHz.
- When the snapshot is more than one block, correlate each block and add the powers. That survives a navigation-bit sign flip and a coarse frequency bin. The cost is sensitivity.
- One crystal driving both the local oscillator and the sample clock couples carrier offset to code rate: a carrier offset `Δf` moves the code, relative to the sample clock, by `Δf / f_c`. Over a long integration the peak walks off one lag. With code-Doppler compensation on, each group of bins uses a replica at chip rate `R_c × (1 + f_g / f_c)`, and the group is kept narrow enough that the leftover drift stays under 0.1 chip. An external mixer breaks this: its oscillator error shifts the intermediate frequency and does not change the code rate, and a high-side local oscillator reverses the sign of Doppler. `snappnt acquire` refuses code-Doppler compensation when the frequency plan has an external `lo_hz`.
- The simulator and a device capture are the same SigMF recording. Only the simulator adds a `truth` annotation: the code phase, the frequency offset, and the C/N0 it used. The acquisition function does not know which source the file came from. Evaluation compares against truth only when the annotation is present.

The product of one acquisition is a code phase and a carrier offset. It is not a position and not a range. One code phase is `clock offset + flight time`. A satellite does not share the receiver clock.

## What has actually been received

The [first sky acquisition](https://h-shiono.github.io/snappnt/results/sky-first-acquisition/) (2026-10-04, GitHub issue #74) is a B210-class recording, processed afterwards. An ESP32 has not acquired NavIC from the sky.

The recording is 32 s at 8 MSa/s, tuned to 2489.9995 MHz, through a 2.4 GHz whip and a Nooelec LaNA. NavIC S-band (2492.028 MHz) sits at +2.0285 MHz in that baseband. No notch and no decimation were applied before `snappnt acquire`. Coherent blocks are 4 ms, so the frequency step is 125 Hz. Two hundred fifty blocks is 1 s of non-coherent integration. Fifty blocks is 0.2 s.

PRN 10 stayed in the +9500 Hz bin in every 1 s window, detection metric 6.9 to 13.8, and it still cleared a 0.2 s integration. PRN 7 stayed in the +6250 Hz bin, metric 2.8 to 5.6. Their code phases walked at the rate the carrier offset predicts (`Δf / 2492.028 MHz × 1.023 Mchip/s`): about 3.90 chip/s and 2.57 chip/s. From 1 s to 27 s the largest miss was 0.004 chip and 0.12 chip, both inside one sample.

The white-noise threshold did not hold on that recording. The default false-alarm probability is `1e-3` over the whole grid, assuming independent cells and white noise. All 14 S-band PRNs, and two L5 codes that are not on the air at S-band, scored above it. Those L5 codes are the noise-only reference, near 1.8. PRN 10 and PRN 7 are claimed because they sit above that floor and because the carrier bin and the code-phase drift stay consistent. PRN 5 became separable only after narrow spectral lines were removed, and that step is not in `snappnt` yet. Between 27 s and 28 s both good PRNs jumped by the same 323.7 chips. That is a sample drop in the recording. The reported C/N0 for PRN 10, about 32–35 dB-Hz, reads several dB below the ICD minimum, and the bias of the estimator at this integration length was not measured. The recording and the receiver location are not published.

## Owning both ends

Between two nodes we schedule, the same correlation measures a delay, because we set the clock the satellite will not share. Four choices do that:

1. **Fit the code to the bank.** Indoor delay spread is hundreds of nanoseconds. One 0.2 ms bank already covers every path in a room. A code shorter than the bank lands entirely inside one capture, and every captured sample contributes. Correlating against one period plus the snapshot is the fallback for a code longer than the bank. Processing gain is still only the samples in the capture.
2. **Put the burst inside the capture.** ESP-SDR stops receiving between snapshots, so a one-shot transmission can miss the window. A node we control either repeats the code for longer than the worst schedule skew, or arms the capture from a shared trigger. The skew has to be smaller than the bank, or the repeat train longer than the skew.
3. **Send a pure pilot.** SnapPnt adds blocks in power because a navigation bit can flip and the frequency bin is coarse. A sounding code has no data bits. The first capture still has to find the carrier offset. Two free-running ESP32 crystals can be tens of kilohertz apart at 2.4 GHz, which is inside SnapPnt's ±30 kHz search. Once that offset is known, later snapshots can be added coherently. Integration time is then however many bursts we choose to stack.
4. **Exchange both ways.** Node A transmits and B records a code phase. Then B transmits and A records one. The two clock offsets cancel. The sum of the delays is the round-trip flight time. One direction stays mixed with the clock. SnapPnt never does this step.

The lag is only as fine as the signal bandwidth and the sample grid. One sample at 80 MSa/s is 12.5 ns, about 3.75 m of path. A Wi-Fi-width signal, near 20 MHz, resolves closer to 15 m per chip. Node-to-node level is high enough to interpolate the correlation peak inside a sample. SnapPnt's acquire command does not interpolate. Its code phase sits on the sample grid. Carrier phase at 2.4 GHz is a finer ruler, and indoors it is mostly multipath, which is the quantity RuView already treats as the channel.

## What this is for RuView

The correlation of a known preamble against a snapshot is the channel impulse response: one delay and one amplitude per path. RuView already sounds the channel in the frequency domain, from CSI on packets it did not design. A snapshot receiver plus a preamble we chose is the same channel in the other domain, with the transmit instant and the capture instant under our control.

The receive side can run on a published ESP-SDR snapshot today. The transmit side should be a signal the radio is already allowed to send. A Wi-Fi preamble is such a signal: the sequence is known, the other node captures a snapshot, and the same FFT correlation produces the impulse response. A custom wideband code is a different radio. SnapPnt did not demonstrate it, and an independent ESP32-S3 experiment ([lozaning/ESP32SDR](https://github.com/lozaning/ESP32SDR)) got on-off and frequency-shift keying onto the air while the raw I/Q replay path did not radiate. Do not plan on arbitrary I/Q transmit from current ESP-SDR firmware.

A round-trip of the first path is a range between two radios. ADR-384 (`docs/adr/ADR-384-external-sensor-evidence-ingest.md`) accepts `uwb_echo` and `radar_range`. This delay is neither of those. It needs its own record if it is ever emitted as `spatial.evidence.v1`. Until that record exists, a capture is a measurement with proof `MEASURED`, and a SnapPnt-style file whose truth annotation holds the simulated delay is proof `SYNTHETIC`. It is not a person and not an occupancy decision.

A single peak over a white-noise threshold is not a detection in the 2.4 GHz band. The sky recording already showed that. Keep a code that is not on the air as the noise reference, and require the carrier bin to stay put and the delay to repeat across several scheduled bursts.

## A first check, if this is worth a spike

Do this on the bench before any firmware change on a node:

1. Transmit a known Wi-Fi preamble from a radio that is already allowed to send it.
2. Capture one ESP-SDR snapshot on a second node, with the capture window covering the preamble. Record the tune, the sample rate, and the schedule skew.
3. Run the SnapPnt correlation, unchanged, against that preamble. Confirm the lag moves by the sample count predicted when a cable delay or a short air path is inserted.
4. Repeat the exchange in the other direction and check that the sum of the two lags tracks the round trip, not either clock.
5. Score the peak against a code that was not transmitted. If that code scores with the real one, the threshold is the sky-test failure and the spike stops.

No position solver, no new evidence type, and no claim of ESP32 sky acquisition. Those are later, and only if this check holds.

## Sound, the shell, and the clock

Ultrasound as the tape measure, two and three sensors for the rectangle and the person, the radio residual once that shell is known, and the drift row applied to samples are in [sound-and-radio-ranging.md](sound-and-radio-ranging.md). This note stays the snapshot receiver.

## Sources

- https://h-shiono.github.io/snappnt/
- https://h-shiono.github.io/snappnt/design/architecture/
- https://h-shiono.github.io/snappnt/results/sky-first-acquisition/
- https://github.com/h-shiono/snappnt
- https://espargos.net/espsdr/
- https://github.com/ESPARGOS/esp-sdr
- https://github.com/lozaning/ESP32SDR
