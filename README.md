# GardenAI — GardenCare Demo

> Battery and solar garden nodes, a home server that decides when to water, and an AI assistant that reads the data and proposes changes for the gardener to approve.

GardenCare is a garden watering and monitoring system built from four parts:

1. **Nodes**: battery-powered sensor and watering nodes. An ESP32-C6 watering node drives a 12 V pump; an nRF52840 sensor node runs on a small solar panel and talks Bluetooth LE only, through an ESP32-S3 gateway.
2. **Server**: a Python service on a home VM. It runs the MQTT broker, stores every reading, pulls weather from Open-Meteo, and answers each node's report with a decision: water or not, and how long to sleep.
3. **Client**: a React + TypeScript web app, also packaged with Capacitor for iOS. It shows readings and history, edits watering strategies, sets nodes up over Bluetooth, and manages firmware releases.
4. **Assistant**: a language model (Claude by default) with tools over the garden's data. It can only *propose* changes; the gardener approves or rejects each one.

All firmware updates over the air: canary nodes first, then every node once the gardener approves, with rollback on the node if a new image fails to start.

<p align="center">
  <img src="docs/deployment/balcony.jpg" alt="Solar-powered nRF52840 sensor node on a balcony railing next to potted blueberries with a drip line" width="360" />
</p>
<p align="center"><em>The nRF52840 sensor node on the balcony railing. Its probe is in the blueberry pot below.</em></p>

---

## The app

<table>
  <tr>
    <td align="center"><img src="docs/screen-shot/home.png" alt="Home: three nodes, the Balcony_nRF node's battery, air and soil readings, and 24-hour charts" width="220" /></td>
    <td align="center"><img src="docs/screen-shot/strategies.png" alt="Strategies: editing a watering strategy with times, soil and rain conditions, pump power and watering time" width="220" /></td>
    <td align="center"><img src="docs/screen-shot/assistant.png" alt="Assistant: chat about the Balcony_nRF node, listing the tools it used" width="220" /></td>
    <td align="center"><img src="docs/screen-shot/firmware-updates.png" alt="Settings: language model choice and the node firmware release list with canary, stable and withdrawn releases" width="220" /></td>
  </tr>
  <tr>
    <td align="center"><em>Home</em></td>
    <td align="center"><em>Strategies</em></td>
    <td align="center"><em>Assistant</em></td>
    <td align="center"><em>Firmware updates</em></td>
  </tr>
</table>

- **Home**: one card per node with its status, battery, air temperature and humidity, soil moisture, soil temperature and conductivity, then charts over 24 hours, 2 days or 7 days. Plant photos from a phone or a camera node sit under the readings.
- **Strategies**: watering rules per node. Times of day, plus optional conditions: soil moisture below a limit, chance of rain below a limit, recent rain, a minimum gap since the last watering, and a watering time scaled by the water lost to evaporation since the last one. The app writes the rule back as a sentence so the gardener can check it.
- **Assistant**: a chat that looks at the garden before it answers. Each answer lists what it looked at (overview, node history, readings, strategies, photos, guides). It keeps notes between conversations, which the gardener can read and correct.
- **Settings**: the language model (Claude, OpenAI, Gemini, OpenRouter, Ollama, or any OpenAI-compatible server) and the firmware release list with Approve, Pause and Withdraw.

---

## Architecture

```
   Phone / browser  (React + TypeScript, Capacitor iOS)
        │  HTTP API                      │  BLE (node setup, local control)
        ▼                                ▼
 ┌─────────────────────────────────────────────┐
 │  GardenCare server  (Python 3.12, home VM)  │
 │  ├─ Embedded MQTT broker                    │
 │  ├─ Decision engine: strategies + weather   │
 │  │    + soil + water balance                │
 │  ├─ Open-Meteo weather, 30 min refresh      │
 │  ├─ Firmware releases: canary → stable      │
 │  ├─ Assistant (LLM tools, propose mode)     │
 │  ├─ MCP endpoints: /mcp, /mcp/firmware      │
 │  └─ SQLite                                  │
 └───────────────┬─────────────────────────────┘
                 │  MQTT over WiFi
     ┌───────────┼──────────────────────┐
     ▼           ▼                      ▼
 Watering node   VerticalFarm     GardenGateway (ESP32-S3)
 ESP32-C6        ESP32, mains     always awake, USB powered
 12 V pump                              │  BLE 5 (1M / Coded PHY)
                                        ▼
                                 NrfSensor nodes (nRF52840)
                                 solar + Li-ion, Zephyr + MCUboot
```

### A wake cycle

A battery node cannot be reached while it sleeps, so the node starts every exchange:

1. Wake on a timer, check the battery (sleep at once if it is critical).
2. Power the sensor rail, read the SHT31 and the RS485 Modbus soil probe, connect WiFi and MQTT.
3. Publish telemetry and wait up to 4 s for the server's answer.
4. The server checks each strategy that is due, in order: soil moisture, coming rain, recent rain, gap since the last watering, water lost since the last watering. The first unmet condition skips the strategy.
5. The answer carries the watering (if any), the next sleep length, the server's clock, a copy of the node's strategies for offline use, and a firmware update offer when one is due.
6. The node runs the pump, publishes the pump event, applies an update if offered, and goes back to deep sleep.

**The node never decides to water on its own while the server is reachable.** If the server is down, the node waters from its local copy of the strategies (soil condition only), keeps its readings and pump events in RTC memory, and sends them at the next successful report so nothing is watered twice.

### Water balance

Strategies can adjust the watering time to the water the pot lost since the last watering: Open-Meteo's reference evaporation (ET0), corrected by 3 % per degree that the node's own air temperature differs from the weather model, minus rain. A typical day's loss (4 mm) gives the set time; the result is limited to 0.25 to 2 times that, at most 120 s.

---

## The assistant

The assistant sits above the rules, not inside the node's 4 s answer. Nodes are always answered by plain decision code; the assistant reads the data and proposes changes to that code's inputs.

- **Tools**: garden overview, per-day node history (every watering and the soil moisture rise in the 3 h after it), readings, weather for the past 3 days and the next hours, strategies, plant photo reports, its earlier proposals, its notes, and gardening guides.
- **Propose mode**: it cannot change anything. It proposes a strategy change, a new or deleted strategy, a wake interval, or one extra watering; the gardener approves each one in the app. Limits are checked in code, not in the prompt: wake interval 5 min to 24 h, extra watering at most 60 s and 3 per node per day, at most 20 waiting proposals. A strategy that changed since the proposal is not overwritten.
- **Plant photos**: each new photo goes to the vision model with the node's readings, its note, and the previous report. It returns a structured report: plant, health, water stress, issues with evidence, growth stage, and confidence. The assistant weighs reports against the readings before acting on them.
- **MCP**: the same tools are served over MCP at `/mcp`, so Claude Code or another MCP client can work with the garden under the same limits. A second endpoint, `/mcp/firmware`, gives a firmware developer agent node logs, core dumps, and release upload.
- **Providers**: Claude, OpenAI, Google Gemini, OpenRouter, Ollama, or any OpenAI-compatible server. Conversations are stored in one format and converted per provider. API keys are never sent back to the client.

### Memory and knowledge

The model starts every conversation with no memory of its own. GardenCare gives it four layers, from short-lived to permanent:

| Layer | What it holds | Where | How the model reaches it |
|---|---|---|---|
| Conversation | The current chat, tool calls and results | SQLite | The last 60 messages are sent each turn, trimmed so they always start at a question from the gardener (never at a dangling tool result). Instructions and tool definitions are prompt-cached. |
| Garden data | Every reading, pump event, offline backlog, weather, strategy, proposal and its outcome, photo report | SQLite, photos as files (180 days) | Tools that summarise rather than dump: per-day node history with each watering and the moisture rise in the 3 h after it, latest readings, weather for the past 3 days and the next hours, earlier proposals so it does not repeat a rejected one. |
| **Notes (LLM wiki)** | Lasting facts: which plant each node waters, the pot and its sun, how fast each node dries in heat, how much a watering raises its moisture, sensor quirks, the gardener's wishes | Markdown files, one per note (`garden`, `nodes/<uid>`, `plants/<plant>`, `topics/<topic>`, `lessons`) | The index of every note (name and title) is in every request; the model reads a note with a tool when it needs it, and writes or updates notes as it learns. |
| **Guides (RAG)** | Gardening books and manuals the gardener uploads (PDF, text, markdown, up to 30 MB) | Text page by page in SQLite, chunked into an FTS5 index (Porter stemming) | `search_guides` returns the best passages with guide and page; `read_guide` reads whole pages around a hit. Answers cite guide and page. |

**The wiki.** The notes are the assistant's long-term memory, written by the model and readable by people. The instructions tell it to keep lasting facts only, not the readings of the day, to keep notes short, and to update a note rather than start another. When it does not know something important (the plant, the pot, sun or shade), it asks and writes the answer down: in the Assistant screenshot it learns that the Balcony_nRF probe is in a blueberry pot and records a note. The gardener can read and correct every note in the app or in any text editor, so its memory never drifts out of sight. Limits are in code: note names are validated paths, at most 200 notes of 20,000 characters each. Over MCP the notes are also served as resources (`note://<name>`).

**RAG without embeddings.** Guide search is keyword retrieval over SQLite FTS5 with stemming, ranked by BM25: no vector database and no second provider for embeddings, so it works the same with every model, including a local Ollama one. The model writes the query, so it can rephrase and search again when the first hits are poor.

**From guides into the wiki.** A background job reads a guide once, about 12,000 characters at a time, and has the model merge what matters to *this* garden into the notes: water needs, moisture ranges turned into numbers the strategies can use, temperature limits, signs of too much or too little water. Each fact carries its citation (guide and page). It reads a note before changing it and keeps what the gardener or the assistant learned from observation, which wins over general advice. The job has only the two note tools, shows progress and the notes it changed, and can be cancelled or undone: undo puts every note back as it was before the import.

**Order of use.** For a care question the assistant checks its notes first, then searches the guides for detail, then looks at the garden data, and cites what it used. Guide text and photos are treated as material, never as instructions, and nothing the model reads can change the garden directly: every change is still a proposal the gardener approves.

---

## Firmware updates over the air

Each release is a signed image for one node type and chip. The server reads type, chip, and version from the image itself, so a release cannot claim to be something it is not.

- **Rollout**: a release goes to canary nodes first, and reaches every node set to update automatically once the gardener approves it (or by itself after a clean canary, if that setting is on).
- **Offer, not push**: the server offers the update in its answer to a report, only when the node looks healthy (battery OK, reports on time, no strategy due now). The node applies it at the end of its wake cycle, after measuring and watering.
- **Rollback**: a new image boots once as a test and must confirm itself by reaching the server. Otherwise the bootloader returns to the old one, and the server pauses the release before it reaches more nodes.
- **Signing**: ESP32 images are signed with ECDSA P-256 (RSA-3072 on the S3); nRF52840 images are MCUboot images with ECDSA P-256. Keys stay on the build machine, outside every repository.
- **BLE nodes**: the gateway downloads the nRF image into PSRAM, checks the MCUboot header and TLVs, and uploads it to the node over BLE (mcumgr SMP), across several of the node's wakes if needed. A cut transfer resumes later.
- **Logs**: every attempt is logged step by step, and nodes upload their boot log and core dump after a notable boot.

The firmware screenshot above shows real releases on the running system: lppc 0.1.5 to 0.1.9 for the ESP32-C6 and GardenGateway 0.1.7 to 0.1.8 for the ESP32-S3, one of them withdrawn.

---

## Hardware

<table>
  <tr>
    <td align="center"><img src="docs/hardware/lppc-node.jpg" alt="Hand-wired ESP32-C6 watering node: 12 V battery pack, diaphragm pump, BTS7960 driver, buck converter, RS485 soil probe" width="420" /></td>
    <td align="center"><img src="docs/hardware/nrf-node.jpg" alt="nRF52840 sensor node: enclosure with solar lid, solar charger with 14500 Li-ion cell, nRF52840 SuperMini, MAX3485, SHT31 and RS485 soil probe" width="420" /></td>
  </tr>
  <tr>
    <td align="center"><em>Watering node (ESP32-C6), hand-wired</em></td>
    <td align="center"><em>Sensor node (nRF52840), solar</em></td>
  </tr>
</table>

### Watering node (`LowPowerPlantCare`, ESP32-C6)

- ESP32-C6 SuperMini on ESP-IDF 5.5, deep sleep between wakes.
- 12 V 4S battery pack, 12 V diaphragm pump driven by a BTS7960 H-bridge at 20 kHz PWM, pump current read back from the driver's sense pin.
- Switched 3.3 V rail (P-MOS and a buck converter) for the pump driver logic, the MAX3485, the water-level switch, and the soil probe, so they draw nothing while the node sleeps. The SHT31 stays on the always-on rail.
- Battery divider switched on its high side: 0 µA in deep sleep.
- RS485 Modbus soil probe: moisture, temperature, conductivity.
- Pump safety in a 100 ms guard during every run: water switch, battery sag, stop requests. A cycle watchdog forces the pump off and the node to sleep if a wake hangs.
- Service mode (long press): stays awake, serves a local test page, and opens the Bluetooth setup service for the app.

### Sensor node (`NrfSensor`, nRF52840) and gateway

- nRF52840 SuperMini on Zephyr / nRF Connect SDK with MCUboot.
- Small solar panel and charger board with one 14500 Li-ion cell, in a weatherproof box. The cell is read through a 1 MΩ / 1 MΩ divider.
- SHT31 and the RS485 probe on a switched rail behind a load switch.
- BLE only. The **GardenGateway** (ESP32-S3 with PSRAM, indoors on USB) scans on 1M and Coded PHY, bonds each node, relays its readings as MQTT telemetry on the node's own topics, and delivers the server's answer at the node's next wake. Adoption and release are done from the app.
- KiCad carrier boards for both nodes are generated from Python scripts (placement and fixed copper by script, the rest by a small grid router), not hand-drawn. Not yet fabricated.

---

## Deployment

<table>
  <tr>
    <td align="center"><img src="docs/deployment/blueberry-probe.jpg" alt="RS485 soil probe and drip emitter in a blueberry pot with bark mulch" width="300" /></td>
    <td align="center"><img src="docs/deployment/pots.jpg" alt="Mint and other balcony pots on drip irrigation" width="400" /></td>
  </tr>
  <tr>
    <td align="center"><em>Soil probe and drip emitter in the blueberry pot</em></td>
    <td align="center"><em>Balcony pots on the drip line</em></td>
  </tr>
</table>

The server runs as a systemd service on a home VM and also serves the web app. The nRF node reports through the gateway from the balcony railing; the drip line waters the pots. The Home screenshot above is the live balcony node.

---

## Testing

- **Server**: 124 pytest tests for the GardenCare service: decision logic, offline backlog, water balance and history, firmware releases and offers, BLE gateway adoption, photos, guides, the LLM adapters, the agent, and the MCP endpoint (checked with the official `mcp` client).
- **Client**: Vitest unit tests for strategy validation, formatting, health, and charts; oxlint; TypeScript type check.
- **Firmware**: a clean build of every app is the automated check. Behaviour is verified on the board through service mode and the test page.

---

## Repository layout

This portfolio repo bundles the GardenCare repos through symlinks under `code/`:

```
code/
├── GardenCareServer/        # Python 3.12 + uv: broker, decisions, HTTP API, assistant, MCP, OTA
├── GardenCareClient/        # Vite + React + TypeScript + Tailwind, Capacitor iOS
├── GardenCareFirmware/      # ESP-IDF apps (lppc, plantcam, gateway) + Zephyr app (nrfsensor), KiCad
└── GardenCareVerticalFarm/  # ESP32 mains node: light, fan, irrigation
```

> The product repos are private (available on request). This repo holds the overview, screenshots, and photos.

---

## Stack

| Part | Stack |
|---|---|
| Watering node | ESP32-C6, ESP-IDF 5.5, FreeRTOS, NimBLE, MQTT, RS485 Modbus, I2C, LEDC PWM, deep sleep, secure-signed OTA with rollback |
| Sensor node | nRF52840, Zephyr / nRF Connect SDK, MCUboot, BLE 5 (Coded PHY), RS485 Modbus, I2C |
| Gateway | ESP32-S3 + PSRAM, ESP-IDF, NimBLE central, mcumgr SMP client, MQTT |
| Server | Python 3.12, uv, asyncio, aiohttp, amqtt (embedded broker), SQLite + FTS5, Open-Meteo, pytest |
| Assistant | Claude API with tool use and prompt caching, OpenAI-compatible adapters, MCP (Streamable HTTP) |
| Client | React, TypeScript, Vite, Tailwind CSS, Capacitor (iOS), Web Bluetooth / `@capacitor-community/bluetooth-le`, Vitest |
| Hardware | KiCad 10 boards generated by Python scripts, wiring PDFs from scripts |

---

## Status

Working system in daily use on a balcony in Auckland: the nRF52840 sensor node behind the ESP32-S3 gateway, the server on a home VM, and the web and iOS app. The ESP32-C6 watering node is hand-wired and runs the same firmware update path. Custom PCBs are designed but not yet fabricated.
