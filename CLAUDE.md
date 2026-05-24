# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A self-hosted Home Assistant + ESPHome stack for a single physical device: the **Waveshare ESP32-P4 Wi-Fi6 Touch LCD 3.4C** ("Ivy"). The device is an HA voice satellite with a media player and an animated LVGL UI (clock, weather, Now Playing, and Ivy avatar animations for listening/processing/replying).

There is essentially **one piece of code that matters**: `esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml` (~1150 lines, all sections in one file — no `!include` / `packages:` split). Everything else is infrastructure or HA runtime state.

## Repo layout

```
docker-compose.yml          ← runs both HA and ESPHome containers on the host network
esphome-config/             ← ESPHome container's /config mount (gitignored except YAML + plans)
  waveshare-p4-touch-lcd-4c-ai-ivy.yaml   ← THE device config
  secrets.yaml              ← WiFi / API keys (gitignored)
  archive/                  ← old configs from earlier Speaker Box build, not used
  Ivy/Assets/               ← LVGL PNG frames for the avatar animations
  plan.md, voice-rebuild-plan.md   ← active design docs, read before architectural changes
ha-config/                  ← HA container's /config mount (gitignored — runtime state, DB, .storage)
config/                     ← older/orphan HA config from a previous run (also gitignored)
workplan.md                 ← active workplan with line-anchored fix list for the device
readme.md                   ← user-facing hardware/feature description for the Ivy device
```

`ha-config/` and `config/` hold the live Home Assistant database, `.storage`, logs, etc. They're gitignored and **not the source of truth** — anything you change there is runtime state, not configuration. The HA `configuration.yaml` itself is essentially stock (`default_config:` plus theme/automation includes); the actual integrations are configured via the HA UI and live in `.storage/`.

## Running things

Both services run via Docker Compose on the host network:

```bash
docker compose up -d              # start HA + ESPHome
docker compose logs -f esphome    # follow ESPHome compile/upload logs
docker compose logs -f homeassistant
docker compose restart esphome
```

To compile/upload the device config, use the ESPHome container's CLI:

```bash
docker compose exec esphome esphome compile waveshare-p4-touch-lcd-4c-ai-ivy.yaml
docker compose exec esphome esphome run     waveshare-p4-touch-lcd-4c-ai-ivy.yaml   # compile + OTA
docker compose exec esphome esphome logs    waveshare-p4-touch-lcd-4c-ai-ivy.yaml   # device logs
```

Add `--no-states` to `esphome logs` to suppress the client-side `[S][media_player]: 'X' >> IDLE` spam — those lines come from `aioesphomeapi`'s `state_log_formatter.py` and **cannot be silenced via the YAML `logger:` block**.

There is no test suite, no lint step, no build system beyond `esphome compile`.

## Architecture: how the device config is structured

The single YAML defines a fully self-contained voice-satellite-with-media-player. Top-level sections (in file order) and what they do:

- **`esphome.on_boot` (priority -100)** — gated boot sequence: WiFi → HA API → audio buffer prime → MWW start → reveal main LVGL screen → start idle timer. Each milestone updates the `boot_label` on the boot screen.
- **`esp32` / sdkconfig** — P4-specific tuning. **Several `sdkconfig_options` are load-bearing** and have block-comment warnings in the file (SDIO streaming mode, L2 cache 256KB, SPIRAM_FETCH_INSTRUCTIONS/RODATA, MBEDTLS_EXTERNAL_MEM_ALLOC). Don't change these without reading the comments — they encode the fix for a specific class of failures (SDIO batching, TLS exhaustion, audio glitches).
- **`esp32_hosted`** — SDIO link to the ESP32-C6 Wi-Fi co-processor.
- **`external_components`** — `n-IA-hane/esphome-intercom` provides `audio_processor`, `i2s_audio_duplex`, and `esp_aec`. This is **not** an upstream ESPHome component; the duplex I²S + hardware AEC depends on this fork.
- **`audio_dac` (ES8311) / `audio_adc` (ES7210) / `esp_aec` / `i2s_audio_duplex`** — the audio stack. The AEC and I²S duplex bus are linked via `processor_id: aec_processor`. `fir_decimator: custom` is required (default SIMD path is broken on P4 RISC-V cores).
- **`microphone:`** — two virtual mics on the same duplex bus: `mic_raw` (pre-AEC, for wake word) and `mic_aec` (post-AEC, for VA / STT). See "Voice pipeline" below.
- **`speaker:`** — `hw_speaker` (I²S out) ← `audio_mixer` (mixes `va_speaker_mix` + `media_speaker_mix`) ← `music_resampler` for media. Mixer/resampler task stacks live in PSRAM.
- **`micro_wake_word`** — two parallel models (`hey_jarvis` + `hey_mycroft`), both at `probability_cutoff: 0.60`, `sliding_window_size: 6`. Runs on `mic_raw`.
- **`voice_assistant`** — the HA Assist client.
- **`script:`** — UI orchestration (`set_anim_state`, `screen_idle_timer`, `update_clock_display`, `refresh_weather_display`, `safe_wake_word_start/stop`, `pipeline_watchdog`, etc.).
- **`display` / `lvgl` / `image` / `font`** — LVGL UI: boot screen, main screen with clock/weather/Now Playing/Ivy avatar. `transform.mirror_x: true` pre-mirrors for an optional Pepper's Ghost hologram enclosure.
- **`number:` / `button:` / `switch:` / `light:`** — entities exposed to HA (Voice Assistant Volume, Music Volume, Screen Brightness, Restart, Enable Wake Word).

## Voice pipeline — the part most likely to change

This is the most fragile and most-edited part of the file. Key invariants to know before touching it:

- **Five LVGL animation states** managed by the `set_anim_state` script: 0=Idle, 1=Listening, 2=Processing, 3=Replying, 4=Now Playing. UI transitions are driven by `voice_assistant.on_listening` / `on_stt_end` / `on_tts_start` / `on_end` / `on_error` and by media-player state callbacks. **The UI should never be in a different state from the actual pipeline** — past bugs have all been variants of "UI says listening, pipeline isn't."
- **Mic routing is deliberate.** `mic_raw` (pre-AEC) feeds MWW; `mic_aec` feeds the voice assistant. The comment in the AEC block explains: with the current `sr_low_cost` AEC mode, MWW-on-raw is the safer choice. There is an active proposal in `esphome-config/voice-rebuild-plan.md` to consolidate onto `mic_aec` and let the VA component own MWW natively — **read that doc before changing mic topology**.
- **`voice_assistant.start` silently no-ops if VA isn't IDLE.** Multiple past bugs trace back to this. If you change the wake/VA orchestration, don't trust a `voice_assistant.start` call to actually do anything — gate it on state or use the native MWW-in-VA integration (per voice-rebuild-plan.md §3).
- **`pipeline_watchdog`** is a band-aid that force-recovers after ~12 s if TTS never starts. The plan is to remove it once the orchestration is fixed properly; don't add new watchdogs as a workaround for new bugs.
- **AEC tuning** (`mode`, `filter_length`, `aec_reference_buffer_ms`) interacts with MWW detection rate. Changing AEC mode without re-testing wake-word accuracy is a regression risk.

## Working with this codebase

- **Single-file edits**: nearly all changes are inline edits to `waveshare-p4-touch-lcd-4c-ai-ivy.yaml`. Treat it like a small codebase in itself — section headers in the file delimit "modules."
- **Comments in the YAML are load-bearing documentation.** Many of them encode the *reason* a setting is the way it is (often: "this was tried, broke X, reverted"). Read nearby comments before changing a setting, and update them when you change the setting.
- **Plan docs are active, not historical.** `workplan.md`, `esphome-config/plan.md`, and `esphome-config/voice-rebuild-plan.md` describe in-flight work with line-anchored references into the YAML. Check them before starting non-trivial changes — there may already be an agreed-on approach.
- **Secrets** are referenced as `!secret name` and live in `esphome-config/secrets.yaml` (gitignored). Don't inline secret values when editing the YAML.
- **Hardware constants** worth knowing: `weather.forecast_home` is the user's HA weather entity (not `weather.home`); the device's friendly name is "WaveShare P4 Touch LCD 4 AI"; the wake words are `hey_jarvis` + `hey_mycroft`.

## Relevant skills

When the user's request fits, reach for these (they're registered globally, not project-bound, but they line up well with this repo):

- **`esphome-code-assistant`** — for substantive ESPHome work: adding/editing components (audio, LVGL, sensors, lights, climate, etc.), lambdas, external components, troubleshooting compile or device errors, HA integration. Don't invoke it for one-line edits; do invoke it for new components or pipeline-level changes.
- **`verify`** / **`run`** — when a change needs to be confirmed on the real device (compile + OTA + watch logs / behavior), not just by reading the diff.
- **`code-review`** / **`review`** / **`security-review`** — for reviewing the current diff or a PR.
