# AGENTS.md — PMCollector

PMCollector is a C++ project. It implements the long-term `Collector`
tier of the Purple Moonlight boat-data architecture; this is the real
collector, not a Python scaffold. (An earlier plan to use Python here
for data-gathering scaffolding was dropped once existing command-line
tools — canboat — covered that need.)

Read the shared guidelines first:
- [General](../general_documentation/Agentic_Coding/AGENTS.md)

## Captured NMEA 2000 data

`captures/` holds a read-only NMEA 2000 dump used as input data for
collector development:

- `*.canboat.txt` — raw canboat-format frames
  (`timestamp,prio,pgn,src,dst,len,bytes…`)
- `*.jsonl.gz` — canboat analyzer-decoded JSON Lines (PGN name + decoded
  fields per message)
- `*.errors.tylersaved` — analyzer parse errors set aside for reference
  (the `.tylersaved` suffix marks files Tyler has manually preserved)

Do not modify or regenerate files in `captures/`. Decompressed working
copies belong in `unzips/` (gitignored).

## Project context

Design decisions, hardware research, and open questions from the prior
codex/ChatGPT sessions live in the Purple Moonlight repo at
`/Users/ai-agent/Documents/workspace/Purple Moonlight`. Read these
before making design-affecting changes here:

- `Purple Moonlight/develop/SYNOPSIS.md`
  — high-signal dump of the full project state (architecture, hardware,
  power, bus, NDA, and open-question context)
- `Purple Moonlight/develop/jason-review/index.md`
  — architecture review draft (Collector / Aggregator / Reporter,
  electrical boundaries, transport, retention contract)
- `Purple Moonlight/develop/jason-review/review-questions.md`
  — questions out to the technical reviewer
- `Purple Moonlight/develop/jason-review/follow-up-for-tyler.md`
  — items still pending Tyler's decision
- `Purple Moonlight/develop/shopping_list.md`
  — current hardware purchases and rationale

This file adds only PMCollector-specific overrides.
