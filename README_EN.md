**[中文](README.md)** | English

# esp-dev-skill

Unified AI Skill for the entire Espressif ecosystem. Aggregates development knowledge from **41** ESP framework/SDK repositories (**554** scenario recipes + API/config/pitfall references) into one installable skill — all content sourced from real documentation and source code.

Supported chips: ESP32, ESP32-S2/S3, ESP32-C2/C3/C5/C6/C61, ESP32-H2/H4/H21, ESP32-P4, ESP8266.

## Features

- Based on real documentation and source code from 41 Espressif official repositories
- Core frameworks: ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK
- Protocol & connectivity: Matter, Zigbee, Thread, BLE (NimBLE), ESP-NOW, MQTT, Modbus, USB
- AI/Voice/Vision: ESP-SR, ESP-DL, ESP-Skainet, ESP-DSP, ESP-Vision, ESP-Who
- Domain frameworks: ADF/GMF audio, Brookesia, Claw, IoT Solution, RainMaker
- Security: mbedTLS, TF-PSA-Crypto, secure certificate management
- Each sub-skill includes recipes (complete call chains), API quick reference, config reference, common pitfalls
- Cross-repository recipe master index (554 scenarios in one table)

## Installation

### 1. Clone to your skill directory

Find or create the skill directory based on your AI Agent documentation:

```bash
git clone <repo-url> esp-dev-skill
```

Or copy the directory directly:

```bash
cp -r esp-dev-skill /path/to/skills/
```

Examples:

> **Claude Code**
> **Project scope**: `.claude/skills` in project root
> **User scope**: `~/.claude/skills` (applies to all projects)
> Navigate to the skills folder and run `git clone <repo-url> esp-dev-skill`

> **QwenCode**
> **Project scope**: `.qwen/skills` in project root
> **User scope**: `~/.qwen/skills` (applies to all projects)

> **OpenCode**
> **Project scope**: `.opencode/skills` in project root
> **User scope**: `~/.config/opencode/skills` (applies to all projects)

### 2. Use the skill

Confirm the skill is loaded in your AI Agent, then invoke it via command.

> **Claude Code**
> Run `claude` in terminal, then type `/esp-dev-skill` and describe your needs

> **QwenCode**
> Run `qwen` in terminal, type `/skills`, select esp-dev-skill

> **OpenCode**
> Run `opencode` in terminal, type `/skills`, select esp-dev-skill

## How It Works

The skill defines a workflow that the AI Agent follows when generating code:

| Step | Name | Description |
|------|------|-------------|
| 1 | Locate | Identify the target framework/SDK/feature, consult the routing table |
| 2 | Enter sub-skill | Open `repos/<repo>/SKILL.md`, read core principles and recipe index |
| 3 | Match recipe | Find the matching scenario in `repos/<repo>/recipes/`, get the complete call chain |
| 4 | Query | Look up API/config/pitfall docs in `repos/<repo>/resources/` |
| 5 | Validate | Verify includes, init order, config macros, pin/chip support |
| 6 | Confirm | Present implementation plan to user (headers, init sequence, main loop, build command) |
| 7 | Execute | Generate code following standard project structure |
| 8 | Build & flash | ESP-IDF: `idf.py build flash monitor` / Arduino: upload / AT: firmware packaging |

## Cross-Repository Principles

- **Choose the right layer first**: ESP-IDF (low-level control) / Arduino-ESP32 (rapid prototyping) / ESP8266 RTOS SDK (ESP8266)
- **Version-sensitive**: ESP-IDF APIs change across major versions — mind the target version
- **Reusable components**: Prefer components from the ESP Component Registry
- **Grounded first**: All content is from real repository documentation — no fabrication
- **Protocol stacks have dependencies**: BLE/Zigbee/Thread require corresponding chip capabilities (classic Bluetooth = ESP32 only, BLE Mesh/Thread/Zigbee = C/H series)
- **Chip capability differences**: Different chips support different peripherals, protocols, and memory — always verify

## Covered Repositories (41)

### Core Frameworks (3)
`esp-idf` · `arduino-esp32` · `ESP8266_RTOS_SDK`

### Domain Frameworks (12)
`esp-adf` · `esp-gmf` · `esp-wdf` · `esp-wasmachine` · `esp-brookesia` · `esp-iot-solution` · `esp-vision` · `esp-who` · `esp-insights` · `esp-claw` · `esp-amp` · `esp-lowcode-matter` · `esp-agents-firmware`

### Protocol & Connectivity (13)
`esp-matter` · `esp-zigbee-sdk` · `esp-thread-br` · `connectedhomeip` · `esp-now` · `esp-hosted-mcu` · `esp-nimble` · `esp-mqtt` · `esp-modbus` · `esp-usb` · `tinyusb` · `esp-at` · `esp-protocols`

### AI / Voice / Vision / DSP (7)
`esp-skainet` · `esp-sr` · `esp-dl` · `esp-dsp` · `esp-detection` · `esp-video-components` · `esp-rainmaker`

### Security / Crypto (3)
`mbedtls` · `TF-PSA-Crypto` · `esp_secure_cert_mgr`

### Other Component Libraries (2)
`esp-bist` · `esp-serial-flasher`

## References

- Cross-repository Recipe Index → `resources/recipe_index.md` (554 scenarios in one table)
- Repository Index → `resources/repo_index.md`
- Glossary → `resources/glossary.md`
- Sub-skill docs → `repos/<repo>/SKILL.md`
- Espressif Official Website → https://espressif.com

## License

Content follows each upstream repository's license (mostly Apache-2.0). This aggregated skill metadata: Apache-2.0.
