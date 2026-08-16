# WaveShare P4 Touch LCD 4C — "Friday" AI Screen

ESPHome config for the **Waveshare ESP32-P4 Wi-Fi6 Touch LCD 3.4C** running a full
voice-assistant + media-player UI with an animated avatar ("Friday").

---

## Hardware

| Component | Detail |
|---|---|
| Main SoC | ESP32-P4 (RISC-V, 400 MHz dual-core, 32 MB flash) |
| Wi-Fi/BLE | ESP32-C6 co-processor, connected via SDIO (esp32_hosted) |
| DAC | ES8311 — 16 kHz, 16-bit, I²C @ 0x18 |
| ADC | ES7210 — 16 kHz, 16-bit, I²C @ 0x40 |
| Display | MIPI-DSI, 3.4" round, LVGL @ 180° rotation |
| PSRAM | 100 MHz (buffers AEC, mixer, LVGL render) |

---

## Features

### Voice Assistant
- **Dual wake-word detection** — `hey_jarvis` + `hey_mycroft` run in parallel via
  `micro_wake_word`. Two different phoneme patterns improve detection for non-native
  accents. Both are tuned with `probability_cutoff: 0.60` and
  `sliding_window_size: 6` (~120 ms sustained confidence) to reject music
  false-triggers.
- Wake word listens on **raw microphone** (`mic_raw`, pre-AEC) — the neural model
  handles speaker echo better on raw audio than on AEC-processed audio.
- STT/TTS runs through **Home Assistant Assist** pipeline.
- **Voice Activity Detection (VAD)** via `vad.json` model runs alongside wake-word.

### Hardware Acoustic Echo Cancellation (AEC)
- Custom `i2s_audio_duplex` component (github: n-IA-hane/intercom-api) provides a
  true full-duplex I²S path with built-in `esp_aec`.
- AEC mode: `voip_high_perf`, reference source: `ring_buffer` with 100 ms buffer
  in PSRAM for accurate delay alignment.
- The FIR decimator uses the `custom` (scalar float) path — the default SIMD
  decimator is unreliable on ESP32-P4 RISC-V cores.

### Media Player
- Streams music from Home Assistant via the `speaker` media player platform.
- Separate mixer channels: `va_speaker_mix` (TTS/announcements) and
  `media_speaker_mix` (music → resampler → mixer → hardware speaker).
- **Two independent volume controls** exposed to HA:
  - *Voice Assistant Volume* — applied during TTS and on idle
  - *Music Volume* — applied when music starts playing
- Format: FLAC for music, WAV for announcements, both at 16 kHz mono.
- Mixer and resampler task stacks live in PSRAM to free internal RAM.

### Animated LVGL UI
Five exclusive screen states, managed by the `set_anim_state` script:

| State | Trigger | Display |
|---|---|---|
| 0 — Idle | Default / VA ends | Clock + date + weather |
| 1 — Listening | Wake word detected | Friday listening animation |
| 2 — Processing | STT end | Friday processing animation |
| 3 — Replying | TTS start | Friday replying animation |
| 4 — Now Playing | Music plays | Song title (scrolling) |

- Friday animations use odd-numbered frames only (1, 3, 5 … 13) for 7-frame sequences
  at 2600 ms per loop — smooth with half the flash cost.
- The `state_anim` widget switches image arrays on each state change.

### Clock & Weather
- Time from Home Assistant (`homeassistant` time platform), updated every minute.
- Weather temperature and condition icon from `weather.forecast_home`.
- Weather icons rendered with Material Design Icons (MDI) webfont at 96 px.
- **Auto-refresh every 5 minutes**: screen wakes briefly (backlight on), shows
  updated clock/weather for 30 s, then dims — only fires when in idle state.

### Screen Backlight
- PWM backlight on GPIO26 (LEDC, 1 kHz, inverted).
- **5-minute idle timer** (`screen_idle_timer`) turns backlight off after no VA
  or touch activity.
- Backlight turns on immediately on wake-word detection.

### Boot Sequence
Priority −100 ensures all hardware is ready before the boot script runs:
1. WiFi connected
2. Home Assistant API linked
3. I²S/SDIO buffers cleared (one MWW cycle)
4. Wake word started
5. Main UI revealed, clock + weather populated

Boot screen shows a spinner + status label at each milestone.

### Pepper's Ghost Hologram (optional)
`transform: mirror_x: true` pre-mirrors the display horizontally so that text and
images read correctly when reflected in a pyramid hologram enclosure.

### Controls exposed to Home Assistant
| Entity | Type | Purpose |
|---|---|---|
| Enable Wake Word | Switch | Toggle wake-word detection on/off |
| Voice Assistant Volume | Number (0.1–1.0) | TTS / idle volume |
| Music Volume | Number (0.1–1.0) | Music playback volume |
| Screen Brightness | Light | Backlight PWM |
| Restart AI Screen | Button | Reboot device |
| Media Player | Media Player | Full HA media control |

---

## File layout

```
esphome-config/
  ha-friday.yaml                           ← main config (this device)
  secrets.yaml                             ← WiFi / API keys (gitignored)
  archive/
    waveshare-main.yaml                    ← archived Speaker Box display config
    waveshare-speaker-box.yaml             ← archived Speaker Box base config
  ha-friday/Assets/
    listening/   listening-1.png … listening-13.png
    processing/  processing-1.png … processing-13.png
    replying/    replying-1.png  … replying-13.png
```

---

## Key constants to tune

| Setting | Location | Default | Notes |
|---|---|---|---|
| Wake-word cutoff | `micro_wake_word.models[*].probability_cutoff` | 0.60 | Lower → more sensitive, higher → fewer false triggers |
| Wake-word window | `micro_wake_word.models[*].sliding_window_size` | 6 | Frames (×20 ms each) the model must sustain confidence |
| Pipeline watchdog | `pipeline_watchdog` delay | 12 s | Time before forced recovery if TTS never starts after STT |
| Screen idle timeout | `screen_idle_timer` delay | 5 min | Time before backlight off |
| AEC reference buffer | `aec_reference_buffer_ms` | 100 ms | Increase if echo leaks through; decrease if voice sounds thin |
