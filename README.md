# Home Assistant & ESPHome Smart Home Stack

This repository contains the configuration, automation, and documentation files for a self-hosted Home Assistant and ESPHome smart home integration stack. It supports two main physical hardware devices: an interactive voice-assistant satellite and a low-power ePaper calendar dashboard.

---

## Workspace Projects

This workspace contains configurations for two primary hardware setups:

### 1. [WaveShare P4 Touch LCD 4C — "Ivy" AI Screen](file:///home/eok/src/home-assistant/docs/ivy-voice-assistant.md)
* **Hardware**: Waveshare ESP32-P4 Wi-Fi6 Touch LCD 3.4C.
* **Features**: Full voice-assistant (Home Assistant Assist) with dual wake-word support, a media player with hardware acoustic echo cancellation (AEC), and an animated LVGL UI featuring an avatar ("Ivy"), weather, and clock displays.
* **Config**: [waveshare-p4-touch-lcd-4c-ai-ivy.yaml](file:///home/eok/src/home-assistant/esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml).
* **Documentation**: See [Ivy Voice Assistant UI Documentation](file:///home/eok/src/home-assistant/docs/ivy-voice-assistant.md) for features, boot sequence details, and key tunable constants.

### 2. [ePaper Calendar + Weather Dashboard](file:///home/eok/src/home-assistant/docs/epaper-calendar-dashboard.md)
* **Hardware**: Seeed Studio XIAO ESP32-S3 + EE02 Driver Board + 13.3" Spectra 6 Display.
* **Features**: A high-resolution (1600x1200) 6-color E-Ink calendar. Wakes up periodically to fetch structured Google Calendar events, countdowns, and weather forecasts from Home Assistant, updates the display, and returns to deep sleep to conserve battery.
* **Config**: [epaper-calendar.yaml](file:///home/eok/src/home-assistant/esphome-config/epaper-calendar.yaml).
* **Documentation**: See [ePaper Calendar Dashboard Documentation](file:///home/eok/src/home-assistant/docs/epaper-calendar-dashboard.md) for full pinouts, software workflow diagrams, and Home Assistant setup steps.

---

## Directory Structure

```
.
├── docs/                             ← Detailed documentation folder
│   ├── ivy-voice-assistant.md        ← Ivy AI Screen Voice Satellite guide
│   └── epaper-calendar-dashboard.md  ← ePaper Calendar Dashboard guide
├── esphome-config/                   ← ESPHome configuration mount
│   ├── waveshare-p4-...-ivy.yaml     ← Ivy voice assistant device configuration
│   ├── epaper-calendar.yaml          ← ePaper calendar device configuration
│   └── secrets.yaml                  ← Device WiFi / API keys (gitignored)
├── ha-config/                        ← Home Assistant configuration mount
│   ├── configuration.yaml            ← HA main configuration
│   ├── automations_epaper.yaml       ← HA calendar & weather automation logic
│   ├── template_sensors_epaper.yaml  ← HA template sensor formatting
│   └── input_text_epaper.yaml        ← HA input helper variables
├── docker-compose.yml                ← Container configuration to run HA & ESPHome
├── CLAUDE.md                         ← Coding guidelines and guidance for the AI assistant
└── workplan.md                       ← Active development workplan
```

---

## Running & Managing Services

Both Home Assistant and ESPHome services are managed via Docker Compose on the host network.

### Compose Commands

To start the containers in the background:
```bash
docker compose up -d
```

To view the service logs:
```bash
docker compose logs -f esphome
docker compose logs -f homeassistant
```

### Compiling & Flashing Devices

To compile and upload configurations to your ESPHome devices, run commands inside the ESPHome container:

* **Ivy Voice Assistant LCD**:
  ```bash
  docker compose exec esphome esphome run esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml
  ```
* **ePaper Calendar Display**:
  ```bash
  docker compose exec esphome esphome run esphome-config/epaper-calendar.yaml
  ```
* **Monitoring Logs**:
  ```bash
  docker compose exec esphome esphome logs esphome-config/epaper-calendar.yaml
  ```
