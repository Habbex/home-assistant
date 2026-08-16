# ESPHome Voice Assistant Pipeline Reference

## Pipeline Architecture

The voice assistant pipeline in ESPHome connects a local device to Home
Assistant's Assist infrastructure for speech-to-text (STT) and text-to-speech
(TTS) processing.

### Component Hierarchy

```
micro_wake_word
  ├── Listens on microphone (mic_aec or mic_raw)
  ├── Runs local TFLite model
  └── on_wake_word_detected → voice_assistant.start

voice_assistant
  ├── Connects to HA Assist pipeline
  ├── Streams audio from microphone to HA for STT
  ├── Receives intent response
  ├── Receives TTS audio URL
  └── Plays TTS via media_player announcement pipeline

media_player (speaker platform)
  ├── announcement_pipeline → va_speaker_mix → audio_mixer → hw_speaker
  └── media_pipeline → media_speaker_mix → audio_mixer → hw_speaker
```

## Micro Wake Word Configuration

### Custom Component Setup

```yaml
  - source:
      type: git
      url: https://github.com/n-IA-hane/esphome-audio-stack
      ref: v2026.7.0
    components: [esp_audio_stack, esp_aec]
  - source:
      type: git
      url: https://github.com/n-IA-hane/esphome-intercom
      ref: v2026.6.2
    components: [audio_processor, speaker]
```

### Model Sources

- **Built-in models**: `hey_jarvis`, `alexa`, `hey_mycroft`, `okay_nabu`
- **Custom models**: URL to a `.json` model descriptor
  ```yaml
  models:
    - model: https://raw.githubusercontent.com/.../model.json
      id: my_model
      probability_cutoff: 0.65
      sliding_window_size: 6
  ```

### Tuning Parameters

| Parameter | Range | Effect |
|---|---|---|
| `probability_cutoff` | 0.0–1.0 | Higher = fewer false triggers, lower sensitivity |
| `sliding_window_size` | 1–10 | Frames (×20ms) model must sustain confidence |
| `gain_factor` | 1–32 | Software gain applied to mic input before model |

### MWW Lifecycle

1. `micro_wake_word.start` — begin listening
2. Detection → `on_wake_word_detected` fires
3. MWW stops itself (default `stop_after_detection: true`)
4. VA pipeline runs
5. On VA `on_end` or `on_error` → restart MWW

**Critical**: MWW must be explicitly restarted after every detection cycle.

## Voice Assistant Callbacks

### Full Callback Chain

```
on_client_connected     → HA connection established
on_client_disconnected  → HA connection lost
on_listening            → STT started, device is recording
on_stt_vad_end          → Voice activity ended (user stopped speaking)
on_stt_end              → STT transcription complete (text available as `x`)
on_intent_start         → HA processing intent
on_intent_progress      → HA processing update
on_tts_start            → TTS text available (text available as `x`)
on_tts_end              → TTS audio URL available
on_end                  → Pipeline complete
on_error                → Pipeline error (code + message available)
```

### Error Handling Pattern

```yaml
on_error:
  then:
    - logger.log:
        format: 'VA error %d: %s'
        args: [code, message.c_str()]
    - voice_assistant.stop:
    - micro_wake_word.start
    - script.execute:
        id: set_anim_state
        target_state: 0
```

## Conversation Context

```yaml
voice_assistant:
  conversation_timeout: 300s  # 5 minutes
```

Retains conversation ID for follow-up commands sharing context.

## Audio Stack Configuration

### Mixer Architecture

```yaml
speaker:
  - platform: esp_audio_stack    # hardware speaker
    id: hw_speaker

  - platform: mixer
    id: audio_mixer
    output_speaker: hw_speaker
    source_speakers:
      - id: va_speaker_mix       # TTS/announcements
      - id: media_speaker_mix    # music
```

### Media Player Pipelines

```yaml
media_player:
  - platform: speaker
    announcement_pipeline:
      speaker: va_speaker_mix
      format: WAV
      sample_rate: 16000
    media_pipeline:
      speaker: media_speaker_mix
      format: WAV
      sample_rate: 16000
```

## ESP32-P4 Audio Specifics

### esp_audio_stack (GMF-based)

Replaces the older `i2s_audio_duplex` component. Embeds codec configuration
directly:

```yaml
esp_audio_stack:
  codec:
    i2c_id: internal_i2c
    input:
      type: es7210
      address: 0x40
      gain_db: 24.0
    output:
      type: es8311
      address: 0x18
      use_mclk: true
```

### Memory Placement

| Setting | Effect |
|---|---|
| `buffers_in_psram: true` | Audio buffers in PSRAM (~15-20KB internal freed) |
| `audio_task_stack_in_psram: false` | Keep audio task stack in internal RAM for speed |
| `task_stack_in_psram: false` | Keep mixer/MWW task stacks in internal RAM |

### AEC Integration

```yaml
esp_aec:
  id: aec_processor
  sample_rate: 16000
  mode: sr_low_cost
  filter_length: 4

esp_audio_stack:
  processor_id: aec_processor
```

The AEC processor is linked to the audio stack via `processor_id`.
