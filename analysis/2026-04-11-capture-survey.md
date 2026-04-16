Got it — clean picture of the bus. **42 distinct devices, 578,856 messages over 36m20s** (02:01:19 → 02:37:55 UTC, 2026-04-11), captured via the Actisense NGT-1 you have on hand (src 0). Here's what's actually on this NMEA 2000 backbone, organized by what it does on the boat.

## How we identify devices

Each device on N2K has a `src` address (0-251). Device identity comes from three handshake PGNs that every device emits at startup and on request:
- **PGN 60928 — ISO Address Claim** → manufacturer, device function, device class
- **PGN 126996 — Product Information** → model ID, software version, serial
- **PGN 126998 — Configuration Information** → installation description (often empty here)

So we know who's on the bus and what each says it is — and we have the per-message counts for everything they emit.

## Sensors (real measurement sources)

| Device | src | What it reports | Count over 36 min |
|---|---|---|---|
| **Airmar DST810 triducer** | 35 | Speed-thru-water, Water Depth, Water Temp (env params), Distance log | 10,953 + 2,197 + 4,390 + 2,186 |
| **WS310 masthead unit** (B&G anemometer) | 44 | Wind Data | 21,252 |
| **Hercules Barometer** | 45 | Actual Pressure | 1,087 |
| **ZG100 Compass** (B&G) | 24 | Vessel Heading, Rate of Turn, Attitude | 21,943 + 21,942 + 2,197 |
| **Precision-9 Compass** (Navico) | 29 | Heading, Rate of Turn, Attitude, Magnetic Field (Simnet) | 43,855 + 43,853 + 21,917 + 21,917 |
| **ZG100 GPS Antenna** (B&G) | 30 | Position rapid, COG/SOG rapid, GNSS Pos, GNSS DOPs, Sats in View, System Time, Mag Variation | 21,921 + 21,920 + 2,196 + 2,196 + 2,180 + 2,197 + 2,196 |
| **Zeus3S iGPS** (MFD-internal) | 31 | Same set as ZG100 GPS, lower rate | similar |
| **Cyclops rig load gateway** | 21 | Load Cell readings (multiple cells via Instance) | 19,708 |
| **Simrad RF25 Rudder feedback** | 43 | Rudder angle | 43,890 |
| **Hercules Boat Speed 1/2/3** | 33,34,36 | Speed (paddle wheel) | ~8,400 each |

## Power / electrical (BEP/Mastervolt-side via N2K)

| Device | src | What it reports |
|---|---|---|
| **MBI** (BEP, Mastervolt Battery Interface) | 20 | Battery Status (9,142), DC Detailed Status (4,571), Inverter Status (2,908), Charger Status (2,906), AC Input/Output (2,182/2,181) — *plus* heavy 65301/65284 manufacturer-proprietary frames |
| **SI** (BEP general sensor box, tank/fluid) | 41 | Fluid Level (3,500) plus 65284/65295 proprietary |
| **COI** (BEP function controller) | 3 | Almost entirely manufacturer-proprietary PGNs (65301/65302/130822/65284/65308) |
| **CZone Display Interfaces ×2** | 49,50 | Manufacturer-proprietary 65284/65294 |

## Navigation processor / MFD stack

| Device | src | What it reports |
|---|---|---|
| **Hercules** (B&G H5000 master sailing processor) | 40 | **Wind Data 35,191** (processed/true), B&G key-value 21,941, Speed 8,429, Sailing Processor Status 2,184, Distance Log 2,171 |
| **Zeus3S Navigator** | 37 | Magnetic Variation, Cross Track Error, Navigation Data, Route/WP, B&G key-value |
| **Zeus3S Echo CH1** (depth/speed channel) | 27 | Environmental Params, Water Depth, Temperature |
| **Zeus3S 12 MFD** | 54 | Mostly manufacturer-proprietary 65280/65288 |

## Autopilot / steering

- **H5000 Pilot Controllers ×3** (src 4,5,6) — Simnet Key Value (~876 each), basically pilot mode/state
- **ZC2 Mode Controller** (src 7) — heavy ISO Request traffic (2,540) — the autopilot computer polling things
- **Zeus3S 12 Pilot Controller** (src 8) — quiet, mostly handshake

## Displays

- **H5000 Graphic Display** (src 46), **Triton2 Instruments ×3** (src 48, 107, 108), **Zeus3S MFD** (src 54)
- These mostly emit handshake + manufacturer-proprietary frames; they're consumers, not sources of physical data

## Gateways / interfaces (Hercules platform — all serial `120996943`)

- **Hercules NMEA0183 1/2** (src 1,2) — bridges from old NMEA 0183 (likely AIS, RPM, etc.)
- **Hercules Analog 1-9** (src 11-19) — **all silent in this capture except for handshake PGNs.** Nine analog input channels and *zero* data emissions. Either nothing's wired to them or they're configured but inactive.
- **Hercules Boat Speed 1/2/3** (src 33,34,36) — speed transducer adapter channels
- **Hercules Barometer** (src 45) — barometer adapter

## Bridges

- **Actisense NGT-1** (src 0) — the capture device itself; emits Actisense status PGNs, not bus data

---

## Thoughts on how to proceed

Aligning this with the SYNOPSIS / jason-review architecture:

1. **The bus is way richer than I'd assume from the design doc's "NMEA-first" framing.** ~85% of the messages are well-known standard PGNs (Wind, Heading, Speed, Position, Rudder, Battery, etc.) — those are decodable today with public canboat tables. The C++ Collector can drink the firehose immediately.

2. **The other ~15% are 65xxx and 130xxx manufacturer-proprietary PGNs** — concentrated on the BEP/Mastervolt power side (CZone) and Simnet/B&G key-value frames. The Cyclops load-cell PGN 65293 is third-party. **This is exactly the territory where Jason's NDA PGN decoder would matter** — keep that NDA boundary in mind when designing the decoder layer (per SYNOPSIS: NDA tables shouldn't go through hosted models).

3. **Redundancy is real**. Two compasses (ZG100 + Precision-9), two GPSes (ZG100 antenna + Zeus3S iGPS), three Hercules speed channels, three Triton2 displays, three pilot controllers. The Collector needs a sender-identity scheme (`src + Model ID + Serial`) so downstream we can tell *which* compass/GPS/speed each reading came from. The design doc lists this as an open item ("Data model, field normalization, and sender / device identification strategy") — this data confirms we need it from day one.

4. **Message rates vary 4 orders of magnitude.** Rate of Turn at ~10 Hz, Heading at ~10 Hz, GPS Pos at ~10 Hz, Wind at ~10 Hz, vs. config/identity stuff at <0.1 Hz. The 5-min/5-MB batching rule from the SYNOPSIS easily holds (~16,000 msg/min ≈ ~3 MB raw / ~0.4 MB compressed).

5. **The 9 silent Hercules Analog channels are a red flag worth asking the boat owner about.** They might be wired but quiet because the source is off, or never wired.

6. **Suggested first C++ collector milestone** based on this data: ingest the canboat-format text stream (or directly via SocketCAN later), parse only the ISO/standard PGNs, key by `(src, pgn, instance)`, durably batch, and surface a per-`src` "device card" derived from 60928/126996/126998. Leave proprietary 65xxx frames as opaque-payload archived bytes for now — that defers the NDA question without losing data.

Want me to dive into any specific device next (e.g., what the Cyclops load cells actually look like, or what fields the Hercules processed Wind Data carries), or move toward a design sketch for the C++ collector ingestion path?
