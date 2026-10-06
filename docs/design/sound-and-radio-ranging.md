# Sound ranging, radio ranging, and the shared clock

**Status:** note for the RuView session. Not an ADR. No evidence type is added here.
**Date:** 2026-10-06
**From:** the WeftOS / cog session, after the snapshot-sounding note. Read that note first for the SnapPnt capture, the sky test, and the bench check. This note is the room: ultrasound for distances, the radio for the person, and the clock that lets the two be compared.

## What each measurement is for

Sound measures the room. A 40 kHz ping crosses a domestic room and stops at the first hard surface. It does not cross a large event hall, for example one 20 m wide by 13 m deep.

The radio follows a person once that room and the sensor poses are known. RuView already consumes the Wi-Fi channel. The missing piece is the shell and the poses, so a path can be named instead of inverted from scratch.

A chirp over a known distance keeps the clocks. The same chirp's direct-path residual is the clock. Its later arrivals, after the shell is subtracted, are the person. Those are two products and they stay separate.

## Sound is the tape measure

Air at room temperature carries sound at about 343 m/s. A 40 kHz cycle is about 8.6 mm, so a bare wall behaves like a mirror. A return is strong when the beam is near straight-on. A grazing wall disappears. One transducer and one ping return a list of ranges down that cone. The first peak is usually the nearest hard surface in the beam. A later peak is either a farther surface or a multipath bounce, and one train cannot say which.

A bat gets a shape because the call is a frequency sweep, the two ears give an angle, and the bat is moving, so each call sees the room from a new place. A fixed 40 kHz burst is one cone and one range profile. The HC-SR04, the sealed JSN-SR04T, and the Marvelmind beacons are already in the WeftOS catalog for that job. Marvelmind's stated figure is about ±2 cm, line of sight, beacon to beacon.

Speed changes by about 0.6 m/s per °C. A 10 °C miss is about 17 cm on a 10 m wall. The BME680 cog can supply the air temperature for that correction. Humidity and drafts still move the path. A 10 m round trip takes about 60 ms, so nodes in one room take turns, or one transmits while the others listen. If they all ping together, each hears the others and reports a false wall at the baseline.

An audible sweep into a quiet microphone is the other way to collect an impulse response. Range resolution of a sweep is `c / (2B)`. A sweep that covers most of the audio band separates surfaces about a centimetre apart, which a short 40 kHz burst cannot. The listener staged for that is the sensiBel SBM100B (catalog chip `sensibel-sbm100b`): optical readout of the diaphragm, 80 dBA signal-to-noise, 14 dBA noise floor, 146 dB SPL overload, 132 dB dynamic range, PDM / I2S / 8-channel TDM, sample rate up to 192 kHz. The published signal-to-noise window is 20 Hz to 20 kHz. The sample clock can carry higher tones, and sensiBel does not claim the diaphragm still responds at 40 kHz. The SBM140B (`sensibel-sbm140b`) is announced at 84 dBA and 136 dB dynamic range, sampling now, production aimed at Q1 2027, full specifications not published. Eight capsules on one TDM clock share a bit clock, so they do not drift against each other inside one node. Wall distances along a beam still want the ultrasonic transducers.

## Two sensors, then three

The placement step already records where each node sits. Sound does not have to discover those poses.

Each sensor stores an empty-room profile first. After the temperature correction, a peak that stays put is the wall on that ray. A sensor against one wall, facing across, measures one dimension from a known point. A second sensor facing the other axis measures the other dimension. Those two numbers are the rectangle. The third sensor is the check: its range has to land on the wall the first two implied. If it falls short, the beam hit furniture, a person, or a wall that is not where a rectangle says it is. An alcove shows up only when some beam actually reaches it.

A person is the peak that is closer than the stored wall and that moves. Two sensors that both see that peak produce two ranges, and two ranges cross at two places. The third range lands on one of them. A sensor that does not see the new peak says the person is outside its cone, which drops the ghost intersection. One cone alone is a distance along that beam and a sector.

The same fix comes from one ping heard by the others. The direct arrival between nodes is the baseline already measured, so that peak is labeled as a node. The wall bounce and the body bounce arrive earlier at the closer sensor. Each pair's time difference is a curve. Two listeners leave the reflector on a curve. Three listeners meet on a point. A shared temperature error moves every absolute range together, so the differences hold a position better than any single range holds the room size. The room size still wants the BME680 correction.

Put the sensors where they look along different directions. Three sensors clustered on one wall learn one distance three times.

## The radio, once the shell is known

The empty room is a fixed set of paths. Each wall is a mirror image of the transmitter, so the direct path and the first bounce off each wall have a length computed from the outline and from where the nodes sit. RuView already fits an empty-room baseline and scores a Fresnel match (`v2/crates/wifi-densepose-signal/src/fresnel.rs`, field `fresnel_confidence`). With the topology filled in, that baseline is predicted: a path that matches the floor plan is the shell, and the residual on those paths is the person. Two links cross in a region. A third link shrinks it. Motion is a change on a path already named.

Knowing the room does not create bandwidth. Delay resolution is about `c / B`. A UWB channel near 500 MHz separates paths by about 0.6 m. A 20 MHz Wi-Fi channel is closer to 15 m. One 80 MSa/s sample is 12.5 ns, about 3.75 m of path, and the analog bandwidth is still the Wi-Fi channel. LoRa at 500 kHz is hundreds of metres. The SX1280's widest chirp, under 2 MHz, still cannot separate a room.

Which radio, for which job:

- **UWB** (802.15.4z, DW3000 class) is the distance instrument between two nodes. Field systems are often 10–30 cm. ADR-384 already accepts `uwb_echo`. The catalog already has the DW1000, DW3000, DWM1001, DWM3000, DW3110, and MDEK1001.
- **Wi-Fi CSI**, and a snapshot of a preamble we already send, stays the room channel. The snapshot technique, the ESP-SDR receive limit, and the bench check are in [snapshot-node-sounding.md](snapshot-node-sounding.md).
- **Bluetooth Channel Sounding** exists on Core 6.0 silicon. The SIG target is about half a metre. A Silicon Labs single-antenna run at 11 m was about ±2–3 m until several antenna paths were combined. An ordinary ESP32 does not implement it. RSSI and direction finding are a different measurement.
- **LoRa** cannot resolve indoor paths.

A cooperative radio delay is neither `uwb_echo` nor `radar_range` until a record is named. Proof is `MEASURED` on a real capture and `SYNTHETIC` when a file's truth annotation holds the simulated delay. A single peak over a white-noise threshold is not a detection in the 2.4 GHz band. Keep an off-air code as the noise reference.

## A known distance turns one chirp into a clock

One code phase is `clock offset + flight time`. A two-way exchange cancels the two offsets and leaves the round trip. The other direction is the useful one once the distance is already measured: the flight is `d / c`, and the residual is the listener's clock error. One chirp is then enough. Two-way exchange remains the check when a distance itself looks wrong.

Use a radio chirp for the clock. Light takes about 10 ns to cross 3 m, and the air-temperature term that moves sound does not move it. The follower pulls its correction so the chirp lands on the predicted instant. Lock the correlator to the first arrival, the direct path, so a stronger wall bounce cannot drag the time base. A step that stays is a node that moved. A slow wander is the crystal. Sound stays the tape measure. A 3 m acoustic path is about 9 ms, which is how the baselines get known. Using the acoustic arrival as the clock would pour the air-temperature error into the time base.

The transmitter's chirp is the master for that slot. Everyone else's time difference equals their distance difference, so the same three-receiver intersection that places a reflector also checks the clocks.

The Epson RX8130CE (catalog chip `epson-rx8130ce`, Akizuki sale code 132109) holds that corrected civil time while the radio is asleep. It is an I2C clock with a factory-adjusted 32.768 kHz crystal, not a time-to-digital converter. One tick is 30.5 µs, about 1 cm of one-way sound and about 5 mm of round-trip range. The echo stopwatch and the chirp timer stay faster counters on the microcontroller. One microsecond of round-trip error is about 0.17 mm of acoustic range.

CSI frames on the current firmware are stamped when the packet arrives at the host, not with a synced node clock (`docs/WITNESS-LOG-110.md` records that the ADR-018 frame has no timestamp field). The chirp correction has to be applied to that stamp before residuals from two nodes can be compared.

## Drift is recorded and applied to the samples

Each board keeps its own counter running. The counter is not steered into being the truth. A crystal on these boards walks by some tens of parts per million, and the rate changes as the part warms. Zeroing the offset at each chirp leaves a ramp between chirps. At 20 ppm that ramp is 20 µs over one second. On the radio, 20 µs is 6 km of path, so an uncorrected rate makes a time difference useless long before the next chirp. On a single ultrasound ping the same rate is harmless. The damage is in comparing nodes.

At every chirp the node appends a row: its local tick, the shared time that chirp solved, and the residual after the known flight is removed. Two rows give the rate. A run of rows gives the curve. The shared solution publishes that row to every node, including the one that transmitted, so one device's walk does not become the room's clock. The message is the anchor time, the offset, and the rate.

Samples stay stamped with the raw local tick. Shared time is:

```
t_shared = t_local − (offset + rate × (t_local − t_anchor))
```

When a new chirp closes the interval, the fit is redone and applied backward across the samples in that interval. Data already in the buffer gets the corrected time when the adjustment is shared. The BME680 temperature estimates the rate between chirps, because that is what bends it. Only the direct-path residual enters the fit. A wall or a person on a reflected path stays out of the clock, or the rate follows the person.

## What may be written as evidence

Wall ranges that agree, each carrying the sensor's known pose, are the room shell. ADR-384 already names `shell_measure` for room geometry that arrives as data. One echo train is not a shell. A label such as `test-room` can validate and still sit in neither world model until `region` resolves through that room's ENU-to-ECEF.

An acoustic range is neither `uwb_echo` nor the optical `tof_depth` record used for the SEN0628. It needs its own type before it is written as `spatial.evidence.v1`. The person's place is the radio residual against the shell, on the corrected clock. Heart rate and breathing stay off the track record and leave as SenML when a sensor actually measures them.

## Parts already staged

Ultrasound modules `hc-sr04`, `jsn-sr04t`, `marvelmind-ultrasonic`. UWB chips and modules DW1000, DW3000, DWM1001, DWM3000, DW3110, MDEK1001. Clock chip `epson-rx8130ce`. Microphones `sensibel-sbm100b` and `sensibel-sbm140b`, beside the existing INMP441. The BME680 cog is the air correction. None of this note adds a cog.

## Sources

- [snapshot-node-sounding.md](snapshot-node-sounding.md)
- ADR-384, `docs/adr/ADR-384-external-sensor-evidence-ingest.md`
- `v2/crates/wifi-densepose-signal/src/fresnel.rs`
- `docs/WITNESS-LOG-110.md` (CSI frame timestamp gap)
- https://www.sensibel.com/products/microphones/sbm100b
- https://www.sensibel.com/news-insights/sensibel-84db-announcement-2026
- https://akizukidenshi.com/catalog/g/g132109/
