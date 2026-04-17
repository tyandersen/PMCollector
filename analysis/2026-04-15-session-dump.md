# Session Dump — 2026-04-15

This file captures the full state of the first PMCollector design
session so a future Claude session (on laptop or elsewhere) can pick
up where we left off.

## Repo setup

- Repo: `git@github.com:tyandersen/PMCollector.git`
- Remote name: `github` (not `origin`)
- Only branch currently: `claude-collector-tortola`
- `develop` will be the main branch later; doesn't exist yet
- Worktree: `PMCollector/claude-collector-tortola/`
- Shared venv: none yet (C++ project, not Python)
- AGENTS.md updated to reflect C++ project, references Purple Moonlight
  design docs

## Project context

PMCollector implements the `Collector` tier of the Purple Moonlight
boat-data architecture. Tyler Andersen is the boat owner (Balance 526
catamaran, *Purple Moonlight*).

Design docs live in a sibling repo:
`/Users/ai-agent/Documents/workspace/Purple Moonlight/develop/`

Key files there:
- `SYNOPSIS.md` — full state dump from prior codex/ChatGPT sessions
- `jason-review/index.md` — architecture review draft
- `jason-review/review-questions.md` — questions for technical reviewer
- `jason-review/follow-up-for-tyler.md` — pending Tyler decisions
- `shopping_list.md` — hardware purchases

The architecture is three tiers:
- **Collector** — near the bus, passive observer, buffers locally,
  tolerates abrupt power loss
- **Aggregator** — onboard hub, pulls from collectors, pushes to cloud,
  hosts telemetry LAN AP
- **Reporter** — cloud system of record (AWS, HTTPS)
- **Client (Shippi)** — iPad app, operator UI for shutdown + status

## Bus scope

Three separate buses, each gets its own collector:
- **NMEA 2000 (N2K)** — easiest; mostly public/standard PGNs; the
  2026-04-11 capture is from this bus
- **MasterBus** (Mastervolt) — separate collector, its own decode
  challenge; some data also appears on N2K via bridge devices (expect
  duplication)
- **Integrel** — separate collector; Jason's NDA PGN decoder is
  specifically for this bus (not N2K, not MasterBus); NDA material
  must never go through hosted AI models

## What's in the repo

```
PMCollector/claude-collector-tortola/
├── .gitignore               (.claude/, unzips/)
├── AGENTS.md                (C++ project, references Purple Moonlight)
├── captures/                (read-only NMEA 2000 dump)
│   ├── 20260411T020035Z.canboat.txt     (44 MB, raw canboat frames)
│   ├── 20260411T020035Z.jsonl.gz        (7.7 MB, decoded JSON Lines)
│   └── 20260411T020035Z.errors.tylersaved (analyzer parse errors)
├── unzips/                  (gitignored, decompressed working copies)
│   └── 20260411T020035Z.jsonl           (126 MB)
└── analysis/
    ├── 2026-04-11-capture-survey.md     (device-by-device bus survey)
    └── 2026-04-15-session-dump.md       (this file)
```

## Capture survey summary

**42 distinct devices, 578,856 messages over 36m20s** (02:01:19 →
02:37:55 UTC, 2026-04-11), captured via Actisense NGT-1.

### Sensors (real measurement sources)

| Device | src | What it reports | Count over 36 min |
|---|---|---|---|
| **Airmar DST810 triducer** | 35 | Speed-thru-water, Water Depth, Water Temp, Distance log | 10,953 + 2,197 + 4,390 + 2,186 |
| **WS310 masthead unit** (B&G anemometer) | 44 | Wind Data | 21,252 |
| **Hercules Barometer** | 45 | Actual Pressure | 1,087 |
| **ZG100 Compass** (B&G) | 24 | Vessel Heading, Rate of Turn, Attitude | 21,943 + 21,942 + 2,197 |
| **Precision-9 Compass** (Navico) | 29 | Heading, Rate of Turn, Attitude, Magnetic Field | 43,855 + 43,853 + 21,917 + 21,917 |
| **ZG100 GPS Antenna** (B&G) | 30 | Position, COG/SOG, GNSS Pos, DOPs, Sats, Time, Mag Var | 21,921 + 21,920 + 2,196 + 2,196 + 2,180 + 2,197 + 2,196 |
| **Zeus3S iGPS** (MFD-internal) | 31 | Same set as ZG100 GPS, lower rate | similar |
| **Cyclops rig load gateway** | 21 | Load Cell readings (multiple cells via Instance) | 19,708 |
| **Simrad RF25 Rudder feedback** | 43 | Rudder angle | 43,890 |
| **Hercules Boat Speed 1/2/3** | 33,34,36 | Speed (paddle wheel) | ~8,400 each |

### Power / electrical (BEP/Mastervolt via N2K)

| Device | src | What it reports |
|---|---|---|
| **MBI** (Mastervolt Battery Interface) | 20 | Battery Status (9,142), DC Detailed (4,571), Inverter (2,908), Charger (2,906), AC In/Out (2,182/2,181) + proprietary |
| **SI** (BEP sensor box) | 41 | Fluid Level (3,500) + proprietary |
| **COI** (BEP function controller) | 3 | Almost all manufacturer-proprietary |
| **CZone Display Interfaces ×2** | 49,50 | Manufacturer-proprietary |

### Navigation processor / MFD stack

| Device | src | What it reports |
|---|---|---|
| **Hercules** (H5000 sailing processor) | 40 | Wind Data 35,191 (processed), B&G key-value 21,941, Speed 8,429, Sailing Status 2,184, Distance 2,171 |
| **Zeus3S Navigator** | 37 | Mag Var, XTE, Nav Data, Route/WP, B&G key-value |
| **Zeus3S Echo CH1** | 27 | Environmental, Water Depth, Temperature |
| **Zeus3S 12 MFD** | 54 | Mostly proprietary |

### Autopilot / steering

- **H5000 Pilot Controllers ×3** (src 4,5,6) — pilot mode/state
- **ZC2 Mode Controller** (src 7) — autopilot computer
- **Zeus3S 12 Pilot Controller** (src 8) — quiet

### Displays

- **H5000 Graphic Display** (src 46), **Triton2 ×3** (src 48, 107,
  108), **Zeus3S MFD** (src 54) — consumers, not sources

### Gateways (Hercules platform — all serial 120996943)

- **Hercules NMEA0183 1/2** (src 1,2) — NMEA 0183 bridges
- **Hercules Analog 1-9** (src 11-19) — **all silent** except handshake
- **Hercules Boat Speed 1/2/3** (src 33,34,36)
- **Hercules Barometer** (src 45)

### Bridge

- **Actisense NGT-1** (src 0) — the capture device itself

## PGN concepts explained

**PGN = Parameter Group Number** from J1939/NMEA 2000. The message-type
identifier embedded in the CAN identifier. It tells you what the data
bytes mean.

**On the wire**: just bytes. Example:
```
2026-04-11T02:01:19.279Z,3,127251,29,255,8,00,9b,80,ff,ff,ff,7f,fd
                       prio  pgn  src dst len  ---- 8 data bytes ----
```

**Decoded** (by looking up PGN 127251 in a decoder table):
```json
{"SID":0, "Rate":-0.058393, "Reserved":"FF 7F FD"}
```

### Decoder file structure

canboat ships `pgns.json` — thousands of PGN definitions. Example for
PGN 130306 (Wind Data):

```json
{
  "PGN": 130306,
  "Id": "windData",
  "Description": "Wind Data",
  "Length": 8,
  "Fields": [
    {"Id": "sid", "Name": "SID", "BitLength": 8, "BitOffset": 0,
     "FieldType": "NUMBER"},
    {"Id": "windSpeed", "Name": "Wind Speed", "BitLength": 16,
     "BitOffset": 8, "Resolution": 0.01, "Unit": "m/s"},
    {"Id": "windAngle", "Name": "Wind Angle", "BitLength": 16,
     "BitOffset": 24, "Resolution": 0.0001, "Unit": "rad"},
    {"Id": "reference", "Name": "Reference", "BitLength": 3,
     "BitOffset": 40, "FieldType": "LOOKUP",
     "LookupEnumeration": "WIND_REFERENCE"}
  ]
}
```

Decoding the bytes `00 6A 05 9C 11 02 FF FF`:

| Field | Bits | Raw | Decoded |
|---|---|---|---|
| SID | 0..7 | 0x00 | 0 |
| Wind Speed | 8..23 | 0x056A = 1386 | 1386 × 0.01 = **13.86 m/s** |
| Wind Angle | 24..39 | 0x119C = 4508 | 4508 × 0.0001 = **0.4508 rad ≈ 25.8°** |
| Reference | 40..42 | 0b010 = 2 | **Apparent** |

### Device identity (MAC equivalent)

NMEA 2000 has a 64-bit **NAME** in PGN 60928 (ISO Address Claim):
- 21-bit Identity Number (manufacturer-assigned unique serial)
- 11-bit Manufacturer Code
- Device Instance, Function, Class, System Instance, Industry Group
- The `src` address (0-251) is **dynamic** — can change on bus restart
- The NAME is persistent — use `(Manufacturer Code, Identity Number)`
  as the canonical device ID

## Architecture decisions from this session

### Decode at the edge

- Don't push raw bytes up the stack and decode in the cloud — that
  defeats the distributed architecture and loses real-time on-boat data
- Collector decodes what it can in C++ using the decoder bundles
- Push structured (decoded) records up for known PGNs
- Push raw bytes + metadata up only for unknown PGNs
- Structured data is graphable immediately; raw is "available for later"

### CPU budget — not a concern

- ~270 msg/s average, ~800 msg/s peak
- Bit-extract + hash-lookup = tens of µs per message in C++
- <5% of one Pi 3 core
- Don't pipe through canboat's `analyzer` at runtime — read raw frames
  directly (NGT-1 binary or SocketCAN), decode in C++ to typed structs

### Decoder bundle layering

Three layers, later overrides earlier for same PGN:
1. **canboat** (public, MIT) — ~700 PGNs, vendored and periodically
   refreshed
2. **integrel-canboat** (NDA) — never leaves the boat, never uploaded
   to cloud, never processed by hosted AI
3. **generated-canboat** — our extensions/proposals, in repo

Distribution: reporter → aggregator → collector (collectors fetch from
aggregator since they may be offline from cloud). Each bundle versioned.
Every decoded record tagged with `decoder_bundle_version`.

### Side-car raw policy

| PGN status | Keep raw bytes? |
|---|---|
| Fully known, no Reserved, stable decoder | drop (decoded is bijective) |
| Known but has Reserved bytes | keep |
| Partially known | keep |
| Provisional / generated-canboat | keep (always) |
| Totally unknown | raw is the only thing |

In practice: **always keeping raw is cheap** (~8-200 bytes/frame) and
enables re-decode against future better decoders.

### AI-assisted decoder workflow

1. Collector flags PGNs it can't decode, persists raw + metadata
2. Aggregator deduplicates: per (manufacturer, pgn, src_name) keeps
   ~100 sample frames + byte-position stats
3. Reporter receives unknown-PGN bundles
4. Cloud-side AI proposes field structures (entropy, stride, correlation
   with known time-series from other PGNs)
5. Proposals land in generated-canboat as `Provisional: true`
6. Tyler reviews + approves/rejects
7. Approved entries promote, bundle version bumps
8. New bundle percolates back down; optional re-decode of archived raw
9. For Integrel (NDA): same loop but stays entirely on-boat

### Actisense NGT-1 / YDNU-02

- NGT-1: proprietary firmware, open USB protocol, open libraries
  (canboat `actisense-serial`)
- YDNU-02 (the "different tool" from shopping list): speaks "Yacht
  Devices RAW" over USB, also supported by canboat
- Both will snarf any CAN-based bus traffic (including Integrel) even
  without knowing what the bytes mean — decode is a separate concern
- C++ collector options: spawn `actisense-serial -r` as subprocess, or
  implement ~200 lines of NGT-1 binary protocol directly

### Observations from the data

- ~85% of messages are well-known standard PGNs — decodable today
- ~15% are manufacturer-proprietary (65xxx, 130xxx) — mostly
  BEP/Mastervolt (CZone) and Simnet/B&G key-value
- Redundancy: 2 compasses, 2 GPSes, 3 speed channels, 3 Triton2
  displays, 3 pilot controllers — need sender-identity scheme from
  day one
- Message rates span 4 orders of magnitude (~10 Hz for heading/wind
  down to <0.1 Hz for config/identity)
- 5-min/5-MB batching rule from SYNOPSIS holds easily (~3 MB raw/min)
- 9 Hercules Analog channels are completely silent (wired but off, or
  never wired?)
- Hercules-platform devices (src 1,2,11-19,33,34,36,40,45) all share
  serial 120996943 — they're virtual sub-devices on one physical box;
  NAME Identity Numbers differentiate them

## Open questions left for Tyler

1. **What's "graphable" vs "available later"?** Graphable = typed
   numeric/enum with units, time-aligned, queryable? Available-later =
   raw bytes + metadata, indexed but no schema?
2. **First analytics question you'd want to answer.** "Wind speed vs
   boat speed" shapes the system differently than "autopilot rudder
   correction frequency over 6 months."
3. **Re-decode on bundle update?** Forward-only, or re-decode historical
   data when decoders improve? (Recommend lazy/on-demand.)
4. **MasterBus↔N2K dedup**: keep both + tag source bus? Prefer one +
   dedup? Prefer one + shadow? (Recommend: keep both, tag source —
   disagreement between two paths is itself a useful signal.)
5. **AI decoder loop location**: cloud for non-NDA, on-boat for
   Integrel NDA? Or uniform?
6. **The 9 silent Hercules Analog channels**: anything wired to them?

## Workflow rules for future sessions

- Remote: `github` (not `origin`)
- Never edit/commit/merge on `develop` or `master`
- Topic branches must not track `develop` as upstream
- Commit format: `claude: <summary>` (or `claude+T:` for joint)
- **Tyler reviews and stages changes; don't commit/push without explicit
  go-ahead**
- Copyright on new code files: `Copyright (c) 2026 Tyler Andersen.
  All rights reserved.`
- Don't chain `cd` with other commands
- Don't browse `~/Documents/workspace/` — stay in the active project
- Shared AGENTS files at
  `~/Documents/workspace/general_documentation/Agentic_Coding/`
