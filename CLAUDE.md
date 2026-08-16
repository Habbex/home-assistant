# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A self-hosted Home Assistant + ESPHome stack managing **two physical devices**:

1. **"Friday"** — a **Waveshare ESP32-P4 Wi-Fi6 Touch LCD 3.4C** voice satellite with media player, animated LVGL UI (clock, weather, Now Playing, and Friday avatar animations for listening/processing/replying), and a custom "hey_friday" wake word.
2. **"ePaper Calendar"** — a **Seeed XIAO ESP32-S3 + EE02 Driver Board + 13.3" Spectra 6 Display** battery-powered wall calendar showing a 7-day timetable grid, weather, forecast, and countdown cards. Wakes every 6 hours, refreshes, and deep-sleeps.

Both services run via Docker Compose (HA + ESPHome containers on host network). The source of truth for each device is a single YAML file in `esphome-config/`.

## Repo layout

```
docker-compose.yml          ← runs both HA and ESPHome containers on the host network
esphome-config/             ← ESPHome container's /config mount (gitignored except YAML + plans)
  ha-friday.yaml            ← Friday: P4 voice satellite config (~1074 lines)
  epaper-calendar.yaml      ← ePaper Calendar: S3 deep-sleep dashboard (~1062 lines)
  secrets.yaml              ← WiFi / API keys (gitignored)
  archive/                  ← old configs from earlier Speaker Box build, not used
  ha-friday/Assets/         ← LVGL PNG frames for the avatar animations
docs/                       ← device documentation
  ha-friday.md              ← Friday hardware, features, tuning constants
  epaper-calendar-dashboard.md ← ePaper hardware, pinout, layout, HA integration, battery tips
ha-config/                  ← HA container's /config mount (gitignored — runtime state, DB, .storage)
workplan.md                 ← active workplan with line-anchored fix list for devices
readme.md                   ← user-facing hardware/feature description
.agents/skills/             ← ESPHome development skill (see "Relevant skills" below)
```

`ha-config/` holds the live Home Assistant database, `.storage`, logs, etc. It's gitignored and **not the source of truth** — anything you change there is runtime state, not configuration. The HA `configuration.yaml` itself is essentially stock (`default_config:` plus theme/automation includes); the actual integrations are configured via the HA UI and live in `.storage/`.

## Running things

Both services run via Docker Compose on the host network:

```bash
docker compose up -d              # start HA + ESPHome
docker compose logs -f esphome    # follow ESPHome compile/upload logs
docker compose logs -f homeassistant
docker compose restart esphome
```

To compile/upload the device configs, use the ESPHome container's CLI:

```bash
# Friday (voice satellite)
docker compose exec esphome esphome compile ha-friday.yaml
docker compose exec esphome esphome run     ha-friday.yaml   # compile + OTA
docker compose exec esphome esphome logs    ha-friday.yaml   # device logs

# ePaper Calendar
docker compose exec esphome esphome compile epaper-calendar.yaml
docker compose exec esphome esphome run     epaper-calendar.yaml
docker compose exec esphome esphome logs    epaper-calendar.yaml
```

Add `--no-states` to `esphome logs` to suppress the client-side `[S][media_player]: 'X' >> IDLE` spam — those lines come from `aioesphomeapi`'s `state_log_formatter.py` and **cannot be silenced via the YAML `logger:` block**.

There is no test suite, no lint step, no build system beyond `esphome compile`.

## Architecture: Friday (ha-friday.yaml)

The voice satellite YAML defines a fully self-contained voice-assistant-with-media-player. For low-level hardware questions (GPIO capabilities, peripheral registers, SDIO host, I2S, PSRAM controller, memory map), consult the ESP32-P4 PDFs in `.agents/skills/esphome-development/references/`:
- `esp32-p4_datasheet_en.pdf` — pinout, electrical characteristics, memory map
- `esp32-p4_technical_reference_manual_en.pdf` — full peripheral register reference

Top-level sections and what they do:

- **`esphome.on_boot` (priority -100)** — gated boot sequence: WiFi → HA API → MWW start → reveal main LVGL screen → start idle timer.
- **`esp32` / sdkconfig** — P4-specific tuning. **Several `sdkconfig_options` are load-bearing** and have block-comment warnings (SDIO streaming mode, L2 cache 256KB, SPIRAM_XIP_FROM_PSRAM, MBEDTLS_EXTERNAL_MEM_ALLOC). Don't change these without reading the comments.
- **`esp32_hosted`** — SDIO link to the ESP32-C6 Wi-Fi co-processor.
- **`external_components`** — `n-IA-hane/esphome-audio-stack` (v2026.7.0) provides `esp_audio_stack` + `esp_aec`; `n-IA-hane/esphome-intercom` (v2026.6.2) provides `audio_processor` + `speaker`. Pinned to specific tags.
- **`esp_aec` / `esp_audio_stack`** — the audio stack with embedded ES8311 DAC + ES7210 ADC codec config. AEC runs in `sr_low_cost` mode.
- **`microphone:`** — `mic_aec` (post-AEC) feeds both MWW and VA.
- **`speaker:`** — `hw_speaker` (I²S out) ← `audio_mixer` (mixes `va_speaker_mix` + `media_speaker_mix`).
- **`micro_wake_word`** — single custom model (`hey_friday`) at `probability_cutoff: 0.65`, `sliding_window_size: 6`. Listens on `mic_aec` with `gain_factor: 8`.
- **`voice_assistant`** — the HA Assist client with `conversation_timeout: 300s`.
- **`script:`** — UI orchestration (`set_anim_state`, `screen_idle_timer`, `update_clock_display`, `refresh_weather_display`).
- **`display` / `lvgl` / `image` / `font`** — LVGL UI: boot screen, main screen with clock/weather/Now Playing/Friday avatar. `transform.mirror_x: true` for optional Pepper's Ghost hologram.
- **`number:` / `button:` / `switch:` / `light:`** — entities exposed to HA (Voice Assistant Volume, Screen Brightness, Restart, Enable Wake Word).

### Voice pipeline — the part most likely to change

Key invariants to know before touching it:

- **Five LVGL animation states** managed by the `set_anim_state` script: 0=Idle, 1=Listening, 2=Processing, 3=Replying, 4=Now Playing. **The UI should never be in a different state from the actual pipeline.**
- **`voice_assistant.start` silently no-ops if VA isn't IDLE.** Gate it on state or use the native MWW-in-VA integration.
- **AEC tuning** (`mode`, `filter_length`) interacts with MWW detection rate.

## Architecture: ePaper Calendar (epaper-calendar.yaml)

The e-paper YAML defines a battery-powered deep-sleep dashboard:

- **`esphome.on_boot` (priority -100)** — battery measurement → WiFi → HA API → data sync → display render → deep sleep.
- **`esp32`** — Seeed XIAO ESP32-S3, Arduino framework, PSRAM octal mode 80 MHz.
- **`external_components`** — `acegallagher/esphome-bigink` provides the `seeed_epaper_spectra6` dual-controller SPI driver.
- **`deep_sleep`** — 6h sleep, 90s max run, GPIO2 wakeup pin.
- **`display` lambda** — C++ rendering code (~500 lines) drawing a 7-day timetable grid (Mon-Sun), weather section, 2-day forecast, countdown cards, and battery footer. Uses 6 colors (Black, White, Red, Yellow, Blue, Green).
- **Data from HA** — calendar events via 8 `input_text` helpers (pipe-delimited format), weather via `weather.forecast_home`, forecast via `input_text.epaper_weather_forecast_data`.
- **Battery management** — GPIO-switched ADC divider, voltage-to-percentage calibration, threshold-based charging detection.

## Working with this codebase

- **Single-file edits**: nearly all changes are inline edits to one of the two device YAML files. Treat each like a small codebase — section headers delimit "modules."
- **Comments in the YAML are load-bearing documentation.** Many encode the *reason* a setting is the way it is (often: "this was tried, broke X, reverted"). Read nearby comments before changing a setting, and update them when you change the setting.
- **Plan docs are active, not historical.** `workplan.md` and docs in `docs/` describe in-flight work. Check them before starting non-trivial changes.
- **Secrets** are referenced as `!secret name` and live in `esphome-config/secrets.yaml` (gitignored). Don't inline secret values.
- **Hardware constants**:
  - Friday: `weather.forecast_home` is the weather entity; friendly name "WaveShare P4 Touch LCD 4 AI"; wake word `hey_friday`; media player entity `media_player.waveshare_p4_touch_lcd_4_ai_media_player_2`.
  - ePaper: friendly name "ePaper Calendar Dashboard"; calendar entity `calendar.familj`; uses `input_text.epaper_calendar_lineN` (N=1-8) and `input_text.epaper_weather_forecast_data`.

## Relevant skills

The project has an ESPHome development skill registered at `.agents/skills/esphome-development/`:

- **`esphome-esp32-development`** — comprehensive skill for ESPHome + ESP32 work: YAML configuration best practices, ESP32-P4/S3 hardware specifics, voice assistant pipelines, LVGL UI design, e-paper display rendering, deep sleep optimization, lambda C++ patterns, external components, and troubleshooting. Includes deep-dive references in `references/` for voice pipeline, LVGL UI, and e-paper/deep-sleep topics. **Invoke it for any substantive ESPHome work** — new components, pipeline changes, display modifications, compile/runtime troubleshooting, or HA integration.

Additionally, the following global skills are useful when applicable:
- **`verify`** / **`run`** — when a change needs to be confirmed on the real device (compile + OTA + watch logs).
- **`code-review`** / **`review`** — for reviewing the current diff or a PR.
