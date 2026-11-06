# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **portfolio / landing repo** for GardenCare. It holds the public `README.md`, app screenshots, and hardware and deployment photos. The product code lives in sibling repos under `/Volumes/MM4X5/Projects/GardenCare/`, surfaced here as symlinks under `code/` (ignored by git). Treat them as read-only from here: work inside the real repo, following its own `CLAUDE.md` / `README.md`.

No build system, no tests, no package manifest at this level.

## Layout

```
README.md                     # GitHub landing page
docs/
├── screen-shot/              # home, strategies, assistant, firmware-updates, camera-node, wiki-rag, vertical-farm, alert-strategy (.png)
├── hardware/                 # lppc-node.jpg, nrf-node.jpg (hand-wired builds), lppc-pcb.jpg, nrf-pcb.jpg (PCBs), plantcam-node.jpg, nrf-node-enclosed.jpg, lppc-enclosure.jpg
└── deployment/               # balcony.jpg, plantcam.jpg (hero pair), blueberry-probe.jpg, pots.jpg
code/                         # symlinks, ignored by git
├── GardenCareServer        → GardenCare/Server
├── GardenCareClient        → GardenCare/Client
└── GardenCareFirmware      → GardenCare/Firmware (was LowPowerPlantCare until 2026-10-23)
```

The `GardenCare*` names under `code/` are the public names used in the README; the targets keep their internal names.

## Keeping the README accurate

The README is condensed from these sources; check them before changing a claim:

- `GardenCare/FEATURES.md` — every feature and its status (built, deployed, in use), test counts
- `GardenCare/README.md` — the parts and where each stands
- `Server/GARDENCARE.md` — decisions, water balance, assistant, photos, guides, MCP, OTA, gateways
- `Firmware/CLAUDE.md`, `doc/OTA.md`, `doc/OTA-BLE.md`, `doc/HARDWARE-*.md` — firmware and hardware

When replacing a screenshot or photo, keep the filename so README links (and the PortfolioHome index, which links the raw GitHub URLs) don't break.

Keep the README short: the user wants it brief. Add a fact only if it earns its line.
