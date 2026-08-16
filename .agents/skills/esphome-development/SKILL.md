---
name: esphome-esp32-development
description: >
  Skills and instructions for developing ESPHome firmware on ESP32 devices.
  Covers YAML configuration best practices, ESP32-P4/S3 hardware specifics,
  voice assistant pipelines, LVGL UI design, e-paper displays, deep sleep
  optimization, lambda C++ patterns, external components, and troubleshooting.
  Activate when working on any .yaml file in esphome-config/, or when the user
  asks about ESPHome, ESP32, LVGL, voice assistant, e-paper, or device firmware.
---

# ESPHome + ESP32 Development Skill

Use this skill when editing ESPHome YAML configs, writing C++ lambdas, adding
components, troubleshooting compile/runtime errors, or designing device UIs.

## Quick Reference

- **Official docs**: https://esphome.io/
- **LVGL docs**: https://esphome.io/components/lvgl/
- **Voice assistant**: https://esphome.io/components/voice_assistant.html
- **Micro wake word**: https://esphome.io/components/micro_wake_word.html
- **Deep sleep**: https://esphome.io/components/deep_sleep.html
- **Display components**: https://esphome.io/components/display/index.html

### ESP32-P4 Hardware Documentation

- **Datasheet**: `references/esp32-p4_datasheet_en.pdf` — pinout, electrical
  characteristics, memory map, package info. Consult for GPIO capabilities,
  voltage levels, and peripheral availability.
- **Technical Reference Manual**: `references/esp32-p4_technical_reference_manual_en.pdf`
  — full peripheral register descriptions, SDIO host/slave, I2S, SPI, I2C,
  MIPI DSI, PSRAM controller, cache, DMA, and interrupt architecture. Consult
  when debugging low-level hardware issues or writing sdkconfig_options.

For additional reference material (voice pipeline, LVGL UI, e-paper/deep sleep),
see the other files in the `references/` subdirectory.

---

## 1. YAML Structure & Best Practices

### Recommended Section Order

Maintain a consistent order in every device YAML:

1. `substitutions` — device name, pin aliases, version tags
2. `esphome` — name, friendly_name, on_boot, platformio_options
3. `esp32` — board, variant, framework, sdkconfig_options
4. `psram` — mode, speed (if applicable)
5. `logger`
6. `api` / `ota` / `wifi`
7. `external_components`
8. Hardware buses (`i2c`, `spi`, `uart`)
9. Audio stack (`audio_dac`, `audio_adc`, `esp_aec`, `i2s_audio`, `microphone`, `speaker`)
10. Core features (`micro_wake_word`, `voice_assistant`, `media_player`)
11. `globals`
12. `script`
13. Sensors, text_sensors, binary_sensors
14. Display, LVGL, fonts, images
15. Controls (`switch`, `button`, `number`, `light`, `select`)
16. Deep sleep / power management

### Secrets Management

- **Never** hardcode WiFi passwords, API keys, or OTA passwords.
- Use `secrets.yaml` with `!secret key_name` references.
- `secrets.yaml` must be in `.gitignore`.

### Comments Are Load-Bearing

Many settings in this repo have comments explaining *why* they are set to a
particular value — often because the alternative was tried and broke something.
**Always read nearby comments before changing a setting, and update them when
you change the setting.**

---

## 2. ESP32 Platform Configuration

### Framework Selection

- **Prefer `esp-idf`** for ESP32 targets — better memory management, native
  PSRAM support, and required for advanced features (voice, audio, LVGL).
- Use `arduino` only when a required library has no IDF equivalent (e.g., some
  e-paper custom components).

### ESP32-P4 Specifics (Friday voice satellite)

The P4 is a RISC-V dual-core without native WiFi/BLE. Key constraints:

> **Hardware reference**: For GPIO pinouts, peripheral capabilities, memory
> map, SDIO host controller details, I2S/I2C register info, and PSRAM
> controller config, consult the ESP32-P4 PDFs in `references/`:
> - `esp32-p4_datasheet_en.pdf` — pinout, electrical specs, memory map
> - `esp32-p4_technical_reference_manual_en.pdf` — peripheral registers, DMA, cache

- **`esp32_hosted`** is required — connects to ESP32-C6 co-processor via SDIO.
- **Do NOT change SDIO frequency** — the C6 slave firmware expects 40 MHz.
- **L2 cache must be 256 KB** (`CONFIG_CACHE_L2_CACHE_256KB: y`) — 128 KB
  causes cold-start audio glitches with MIPI DSI + audio + LVGL.
- **XIP from PSRAM** (`CONFIG_SPIRAM_XIP_FROM_PSRAM: y`) — the P4-correct
  replacement for the old S3 `SPIRAM_FETCH_INSTRUCTIONS/RODATA` options.
- **TLS in PSRAM** (`CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC: y`) — prevents
  internal SRAM exhaustion after 2-3 HTTPS sessions.
- **`network.enable_high_performance: false`** — prevents SDIO buffer starvation.

### ESP32-S3 Specifics (ePaper Calendar)

- Uses Arduino framework (required by the `esphome-bigink` custom component).
- PSRAM in **octal mode at 80 MHz**.
- `board_build.arduino.memory_type: qio_opi` in platformio_options.
- Deep sleep is the primary power management strategy.

---

## 3. External Components

### Version Pinning

**Always pin external components to a specific tag or commit hash.**
Floating on `main` lets upstream churn break the device on every flash.

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/author/repo
      ref: v1.2.3          # tag or commit hash
    components: [component_name]
```

### Troubleshooting

- **"Component not found"**: Check that the repo has the expected directory
  structure (`components/component_name/__init__.py`). If components live in
  a subdirectory, use the `path:` option.
- **Version mismatches**: After updating ESPHome, verify that pinned external
  components are compatible with the new ESPHome version.

---

## 4. Audio Stack & Voice Pipeline

### Architecture (ESP32-P4 voice satellite)

```
mic_aec (post-AEC) ──→ micro_wake_word (local detection)
                   └──→ voice_assistant (STT/TTS via HA Assist)

hw_speaker ←── audio_mixer ←── va_speaker_mix (TTS/announcements)
                           └── media_speaker_mix (music → resampler)
```

### Key Invariants

1. **MWW stops itself after detection** (`stop_after_detection: true`) —
   restart it from `voice_assistant.on_end` / `on_error`.
2. **`voice_assistant.start` silently no-ops if VA isn't IDLE** — gate on
   state or use native MWW-in-VA integration.
3. **AEC mode affects wake-word accuracy** — `sr_low_cost` preserves spectral
   features for MWW; `voip_high_perf` drops detection rate significantly.
4. **Mic routing is deliberate** — understand which mic stream feeds which
   component before making changes.

### AEC Tuning Parameters

| Parameter | Effect |
|---|---|
| `mode: sr_low_cost` | Linear-only AEC, preserves MWW features |
| `mode: voip_high_perf` | Adds Residual Echo Suppressor, kills MWW |
| `filter_length: 4` | 128 ms echo tail (4 × 32 ms frame) |
| `aec_reference_buffer_ms` | Increase if echo leaks; decrease if voice thins |

### Voice Assistant Callbacks

```yaml
on_listening:       # STT has started listening → show "listening" UI
on_stt_vad_end:     # Voice activity ended → show "processing" UI
on_stt_end:         # Transcription complete → show transcript
on_tts_start:       # TTS audio starting → show "replying" UI
on_end:             # Pipeline complete → return to idle, restart MWW
on_error:           # Pipeline error → cleanup, return to idle, restart MWW
```

---

## 5. LVGL UI Design

### Configuration

- **Display `update_interval: never`** — LVGL manages its own rendering.
- **`auto_clear_enabled: false`** on the display component.
- **`buffer_size: 10-25%`** — lower with PSRAM, higher without.
- Use `rotation:` in the `lvgl:` block (uses P4 PPA hardware accelerator).

### Widget Patterns

- **Labels**: Update dynamically with `lvgl.label.update` in scripts or
  callbacks. Each label supports only one font.
- **AnimImg**: Use for state-based animations. Switch `src` array on state
  change. Keep frame count low to save flash (use every-other-frame strategy).
- **Spinner**: Useful for boot/loading screens. Control visibility with
  `lvgl.widget.update` + `hidden: true/false`.
- **Obj containers**: Use as screen regions. Toggle `hidden` to switch between
  mutually exclusive screens.

### State Machine Pattern

This repo uses a global `current_anim_state` variable with a `set_anim_state`
script that:
1. Guards against no-op transitions (skip if already in target state)
2. Hides all screens
3. Shows the target screen
4. Updates animation sources if applicable

**The UI must never be in a different state from the actual pipeline.**

### Font Optimization

- Define only needed `glyphs` to save flash memory.
- Use Google Fonts via `gfonts://FontName` syntax.
- MDI icons via the MaterialDesign-Webfont TTF with explicit Unicode codepoints.

---

## 6. E-Paper Display Development

### Deep Sleep Lifecycle

```
Boot → WiFi → HA API → Sync Data → Render Display → Deep Sleep
```

- `update_interval: never` — trigger display update manually.
- Allow sufficient `delay` after `component.update` for the e-paper refresh
  to complete physically before entering deep sleep.
- Set `run_duration` as a watchdog to prevent the device from staying awake
  indefinitely if something hangs.

### Battery Optimization

- **`fast_connect: true`** — skip WiFi scanning.
- **Remove captive portal/fallback AP** — they drain battery if WiFi fails.
- **Switch off ADC divider** when not measuring — use a GPIO switch.
- **Consider static IP** — saves DHCP negotiation time.
- **Reduce WiFi TX power** if close to router.

### Rendering Best Practices

- Pre-define color constants for the display's native palette.
- Use layout constants (margins, column widths) at the top of the lambda.
- Helper lambdas for repeated operations (UTF-8 truncation, time formatting,
  weather icon lookup).
- Keep lambda code modular with inline C++ lambda functions.

### Spectra 6 Color Mapping

| Constant | Color | Typical Use |
|---|---|---|
| `Color(0,0,0)` | BLACK | Text, borders |
| `Color(255,255,255)` | WHITE | Background |
| `Color(255,0,0)` | RED | Today highlight, alerts |
| `Color(0,255,0)` | GREEN | All-day events, healthy |
| `Color(0,0,255)` | BLUE | Timed events, icons |
| `Color(255,255,0)` | YELLOW | Warnings, medium battery |

---

## 7. Lambda C++ Patterns

### Common Patterns

```cpp
// Access component state
float temp = id(ha_temperature).state;
bool connected = id(va_engine).is_connected();

// Null/NaN checks (always do this for HA sensors)
if (std::isnan(x)) return std::string("-- °C");

// String formatting
char buf[32];
snprintf(buf, sizeof(buf), "%.1f°C", temp);
return std::string(buf);

// Static variables for persistence across calls
static int counter = 0;
counter++;

// Access time
auto t = id(ha_time).now();
if (!t.is_valid()) return std::string("--:--");
```

### Debugging

- **Check generated code**: `<node_name>/src/main.cpp` shows exact C++ output.
- **Use `logger.log`** with format strings and args for runtime debugging.
- **Heap monitoring**: `esp_get_free_heap_size()` and
  `heap_caps_get_free_size(MALLOC_CAP_SPIRAM)` to track memory.
- **Minimal reproduction**: Comment out YAML sections to isolate compile errors.

### Common Pitfalls

- Referencing an `id()` that hasn't been declared yet.
- Missing `#include` for standard library functions (usually auto-included).
- Blocking operations in lambdas — keep logic non-blocking.
- Using SIMD-optimized paths on RISC-V cores (P4) — use scalar alternatives.

---

## 8. Compile & Deploy Workflow

### Using Docker Compose (this repo's setup)

```bash
# Compile only (validate changes)
docker compose exec esphome esphome compile <config>.yaml

# Compile + OTA upload to device
docker compose exec esphome esphome run <config>.yaml

# Watch device logs
docker compose exec esphome esphome logs <config>.yaml

# Suppress client-side state spam
docker compose exec esphome esphome logs --no-states <config>.yaml
```

### Troubleshooting Compile Errors

1. Read the **full error output** — the relevant line is usually near the end.
2. Check the **generated `main.cpp`** for C++ syntax issues in lambdas.
3. Verify **external component compatibility** with current ESPHome version.
4. Check **PSRAM/RAM constraints** — "Killed" during compile = OOM on host.
5. Verify **sdkconfig_options** — wrong options silently ignored on wrong chip.

### OTA Considerations

- Ensure device is reachable on the network before OTA.
- For ESP32-P4 with hosted WiFi, OTA may be slower due to SDIO bottleneck.
- First flash typically requires USB; subsequent updates use OTA.

---

## 9. Home Assistant Integration

### Entity Naming

- ESPHome auto-generates entity IDs from `friendly_name` + component `name`.
- Verify entity IDs in HA → Settings → Devices & Services.
- Common pattern: `media_player.waveshare_p4_touch_lcd_4_ai_media_player`.

### Data Flow Patterns

```yaml
# HA → Device: sensor/text_sensor with entity_id
sensor:
  - platform: homeassistant
    id: ha_temperature
    entity_id: weather.forecast_home
    attribute: temperature

# Device → HA: entities auto-exposed via API
number:
  - platform: template
    name: "My Control"
    # Automatically appears in HA
```

### Weather Entity

- `weather.forecast_home` is the user's weather entity (not `weather.home`).
- State = condition string (`sunny`, `cloudy`, etc.).
- Attributes = `temperature`, `humidity`, `wind_speed`.

---

## 10. Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Black screen, no logs | Wrong SDIO frequency | Keep default 40 MHz |
| Audio glitch on cold start | L2 cache too small | `CONFIG_CACHE_L2_CACHE_256KB: y` |
| TLS/HTTPS failures after 2-3 sessions | MbedTLS in internal SRAM | `CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC: y` |
| Wake word unreliable | AEC mode too aggressive | Use `sr_low_cost` for MWW |
| `voice_assistant.start` does nothing | VA not in IDLE state | Gate on state check |
| SDIO buffer assertion | `enable_high_performance: true` | Set to `false` |
| Display not updating (LVGL) | Manual `update_interval` set | Use `update_interval: never` |
| E-paper blank after wake | Sleep entered before refresh done | Add sufficient delay |
| MWW re-triggers during TTS | MWW on raw mic picks up speaker | Use post-AEC mic or stop MWW during TTS |
| Heap declining across VA cycles | Memory leak or fragmentation | Monitor with `debug:` sensors |
