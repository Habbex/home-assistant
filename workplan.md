# Workplan: Fix 5 issues with Waveshare P4 AI Voice Satellite

## Context

The user has a Waveshare P4 Touch LCD AI device running ESPHome ([esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml)) acting as a Home Assistant voice satellite with an LVGL UI ("Ivy" assistant + clock/weather + Now Playing screen). After a previous round of changes, five regressions/limitations remain:

1. **Wake word is unreliable** — sometimes responds, sometimes ignores wake words
2. **Weather doesn't show** — user's HA entity is `weather.forecast_home`, but config still references `weather.home` for temperature
3. **Whole pipeline feels sluggish** — multi-second delays accumulate across boot, wake-word detection, and VA recovery
4. **Log spam** — media-player state and volume changes flood the log even with `media_player: WARN`
5. **No music continuation** — wake word `stop`s music permanently; the user wants pause-on-wake then resume-on-timeout (so commands interrupt music briefly, then it picks back up)

The plan below addresses each issue with the minimum diff to the existing YAML and explains *why* each change is safe in light of the prior SDIO/AEC stability work that the comments in the file warn about.

---

## Critical files

- [esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml) — single file, all changes happen here

No other files need to be touched. The archive YAML in `esphome-config/archive/` is unrelated.

---

## Fix 1 — Weather entity ID (quick win, do first)

**Problem:** [esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml:250](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L250) sets `ha_temperature` to `entity_id: weather.home`. The user's actual HA entity is `weather.forecast_home`. The icon (`ha_weather_condition`) on [line 289](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L289) is already correct, which is why some weather data may have appeared and other parts didn't.

**Change:**
```yaml
# line 250
- platform: homeassistant
  id: ha_temperature
  entity_id: weather.forecast_home   # was: weather.home
  attribute: temperature
```

That's it for weather. Both temperature (attribute) and condition (state) come from the same entity. ESPHome's `homeassistant` text_sensor without `attribute:` returns the entity state, which for a HA weather entity is the condition string ("sunny", "cloudy", etc.) — already used correctly.

---

## Fix 2 — Wake word reliability + barge-in during music

**Root cause (two separate issues feeding each other):**

1. `micro_wake_word` listens on **`mic_raw`** ([line 398](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L398)) — the *pre-AEC* mic. With music playing through the speaker, the raw mic picks up the speaker output → false triggers → the existing comment block on line 327–330 explains this is exactly why wake word has to be **stopped** during music.
2. After every interaction, `restart_wakeword_safely` ([line 737](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L737)) waits **4 seconds** before re-enabling wake word. During those 4 s the device is deaf to the wake word. Combined with the `on_end` 800 ms pre-delay ([line 539](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L539)), the user has a ~5 s blind window after every reply.

**Change A — switch wake word to the AEC-cleaned mic stream:**
```yaml
micro_wake_word:
  id: my_wake_word
  microphone:
    microphone: mic_aec        # was: mic_raw — AEC subtracts speaker output
    gain_factor: 8             # was: 6 — AEC reduces level slightly, compensate
  ...
```

The AEC pipeline (`esp_aec` block at line 151) already cancels speaker echo. Routing the wake word through `mic_aec` removes the root cause that the prior workaround (stopping wake word during music) was hiding.

**Change B — shorten the deafness window:**
```yaml
# restart_wakeword_safely (line 737)
- id: restart_wakeword_safely
  mode: restart
  then:
    - delay: 1s          # was: 4s
    ...
```

```yaml
# voice_assistant on_end (line 538-540)
- delay: 300ms           # was: 800ms — pure SDIO settle, with mic_aec less needed
- script.execute: restart_wakeword_safely
```

**Change C — keep wake word running during music (now safe with AEC):**
Remove the `safe_wake_word_stop` from `on_play` ([line 339](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L339)). With AEC cancelling speaker output from the mic stream, music spill no longer triggers the wake word.

**Change D — slight cutoff bump for safety once AEC is on the wake-word path:**
Raise `probability_cutoff` from `0.38` → `0.42` for both models and VAD ([line 403, 406, 410](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L403)). With AEC-cleaned audio the SNR is better, so a slightly higher cutoff still triggers reliably but rejects more music/speech background. **Tune this last** — if wake word becomes harder to trigger, drop back to 0.40.

---

## Fix 3 — Speed up the pipeline

Cumulative latency budget today vs. after:

| Phase | Current | Target | Notes |
|---|---|---|---|
| Boot delays (lines 11, 19, 27, 38, 45) | ~5.0 s | ~2.5 s | Cosmetic LVGL transitions only — halve them all |
| Wake-word → VA listening ([427, 438](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L427)) | 1200 ms drain + 200 ms = **1.4 s** | 600 ms + 100 ms = **0.7 s** | The 1200 ms drain was for SDIO bus stability when stopping a *playing* media stream; 600 ms is the sweet spot per ESPHome speaker-media-player issue threads |
| VA end → wake word ([539](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L539), [740](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L740)) | 800 ms + 4000 ms = **4.8 s** | 300 ms + 1000 ms = **1.3 s** | Already covered above |
| `pipeline_watchdog` ([765](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L765)) | 10 s | 6 s | STT timeout — 10 s is conservative |

Concrete edits:

```yaml
# on_boot delays — halve each cosmetic delay
- delay: 400ms   # was 800ms (3 occurrences: lines 19, 27, 38)
- delay: 800ms   # was 1.5s (line 45)

# on_wake_word_detected (line 427)
- delay: 600ms   # was 1200ms — SDIO RX→TX drain
- voice_assistant.stop
...
- delay: 100ms   # was 200ms (line 438)
- voice_assistant.start: ...

# pipeline_watchdog (line 765)
- delay: 6s      # was 10s
```

Boot is purely cosmetic (LVGL label updates) — no risk in trimming. The 600 ms drain is the part that needs verification on hardware: if the user sees "AI screen goes blank" or boot hangs after a wake word, the safe rollback is 800–900 ms.

---

## Fix 4 — Quiet the logs

> **2026-04-29 update — `[S][media_player]:` lines are CLIENT-SIDE, not device-side.**
>
> After the first round of edits the user still saw lines like:
> ```
> [01:13:43.353][S][media_player]: 'Media Player' >> IDLE
> [01:13:43.353][S][media_player]:   Volume: 75%
> ```
> Source confirmed: `aioesphomeapi/state_log_formatter.py` (`_format_media_player`, lines 253–262 in v44.16.1) generates these from API entity-state messages received by the `esphome logs` viewer / dashboard. **No `logger:` setting on the device can suppress them** — the device just publishes the state, the viewer prints `[S]…`.
>
> Two ways to silence:
> 1. **CLI viewer flag** (definitive): `esphome logs --no-states /config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml`. This sets `subscribe_states=False` so the dashboard never receives state updates to format. Confirmed in 2026.4.2 (`esphome/__main__.py`). The dashboard "Logs" tab does not currently expose a UI toggle, so use the CLI.
> 2. **Reduce the *frequency* of state pushes** — every `media_player.volume_set` call publishes a new state, even when the value is unchanged. The original YAML called `volume_set` unconditionally in `on_idle`, `on_tts_start`, and `on_end`, producing an `[S]` block on every VA cycle. Wrapping each call in a `> 0.04` diff check (mirroring the slider on_value pattern) eliminates the redundant pushes. Also removed a duplicate unconditional `volume_set` at the top of `on_idle` that was followed by an identical conditional one.
>
> Both changes are now applied. The CLI flag is the final answer for users who don't want to see entity state in their logs at all; the diff checks reduce churn (and pointless work) even when the user *does* watch states.

**Root cause (device-side, original analysis still valid):** Only `media_player: WARN` was filtered. Sibling component tags also need silencing — `audio`, `speaker_mixer` (note: actual tag is `speaker_mixer`, not `mixer_speaker`), `resampler_speaker`, `speaker_media_player`, plus the volume slider's `number` template fires INFO each time HA pushes a state update.

**Change to the `logger:` block ([line 68](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L68)):**
```yaml
logger:
  level: INFO
  hardware_uart: UART0
  logs:
    # Standard ESPHome 2026.4.x audio/media tags (verified against
    # /esphome/esphome/components/**/*.cpp source TAG constants)
    media_player: WARN
    speaker_media_player: WARN
    speaker_media_player.pipeline: WARN
    audio: WARN
    audio_reader: WARN
    audio.decoder: WARN
    speaker_mixer: WARN          # tag is speaker_mixer, NOT mixer_speaker
    resampler_speaker: WARN
    speaker: WARN
    # External components (intercom-api: i2s_audio_duplex, esp_aec)
    i2s_audio: WARN
    i2s_audio.speaker: WARN
    i2s_audio.microphone: WARN
    i2s_duplex: WARN
    i2s_duplex.spk: WARN
    i2s_duplex.mic: WARN
    audio_dac: WARN
    audio_adc: WARN
    esp_aec: WARN
    number: WARN                 # silences volume-slider state spam
    homeassistant.text_sensor: WARN
    light: WARN
    sensor: WARN
```

**Also remove these always-on `logger.log:` calls** that fire on every interaction (replace with nothing — keep the surrounding action):
- [line 332](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L332) `"Media player entered PLAYING"`
- [line 364](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L364) `"Media player idle — restoring idle screen"`
- [line 484, 488](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L484) VA connect/disconnect chatter
- [line 498](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L498) `"Mic is listening"` (state animation already shows this)

Keep the diagnostic ones (`Wake word detected: X`, `STT heard: X`, `TTS text: X`, `VA error N: X`, watchdog fires).

---

## Fix 5 — Pause music on wake word, resume on timeout

**Current broken flow:**
- Wake word → `media_player.stop` ([line 425](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L425)) — destroys the stream
- `was_playing_media` global is *set* ([line 419](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L419)) but **never read** anywhere → music never resumes

**Desired flow:**
1. Wake word detected while music is playing → **pause** (not stop), remember it was playing
2. VA cycle runs (listen → process → reply)
3. After VA ends:
   - If user *did* speak a command (we got STT/TTS) → leave music paused, return to clock screen
   - If user *did not* speak (timeout, no STT) → resume music, return to Now Playing screen

**Implementation:**

Add a second global to track whether STT actually fired this cycle:
```yaml
globals:
  ...
  - id: stt_fired_this_cycle
    type: bool
    restore_value: false
    initial_value: 'false'
```

Set it true in `voice_assistant.on_stt_end` ([line 503](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L503)):
```yaml
on_stt_end:
  - globals.set: { id: stt_fired_this_cycle, value: 'true' }
  - script.execute: pipeline_watchdog
  - logger.log: ...
  ...
```

Reset it in `voice_assistant.on_listening` ([line 497](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L497)) so each cycle starts clean:
```yaml
on_listening:
  - globals.set: { id: stt_fired_this_cycle, value: 'false' }
  ...
```

Change `on_wake_word_detected` to **pause** instead of **stop** ([line 425](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L425)):
```yaml
- if:
    condition:
      media_player.is_playing:
        id: board_player
    then:
      - media_player.pause: board_player   # was: media_player.stop
      - delay: 600ms                        # was: 1200ms — covered in Fix 3
```

Add resume logic in `voice_assistant.on_end` ([line 527](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L527)) — *before* the existing `set_anim_state target_state: 0`:
```yaml
on_end:
  - script.execute: screen_idle_timer
  - globals.set: { id: va_speaking, value: 'false' }
  - media_player.volume_set: ...
  # NEW: resume music if user didn't actually issue a command
  - if:
      condition:
        lambda: 'return id(was_playing_media) && !id(stt_fired_this_cycle);'
      then:
        - logger.log: "No command — resuming paused music"
        - media_player.play: board_player
        - script.execute: { id: set_anim_state, target_state: 4 }
      else:
        - script.execute: { id: set_anim_state, target_state: 0 }
  - globals.set: { id: was_playing_media, value: 'false' }
  - delay: 300ms
  - script.execute: restart_wakeword_safely
```

Replace the existing `set_anim_state target_state: 0` block (lines 535-537) with the conditional above.

Verify `media_player.play` (with no `media_url`) actually resumes a paused stream on the speaker platform — per ESPHome speaker-media-player docs the `play` action resumes when paused. If that ever regresses, swap to `media_player.pause_resume` (toggles based on current state).

**Edge case:** if the user *does* say a command like "play the news," the new media stream will start automatically when HA pushes it; the `was_playing_media` flag is reset, so no extra resume happens.

---

## Fix 6 — Snappier Gemini replies (added 2026-04-29)

User reported the Gemini-backed VA replies feel slow and not snappy. Three device-side levers (HA-side toggles separately):

- **`announcement_pipeline` `format: FLAC` → `WAV`** ([line 362](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L362)). FLAC requires the ESP to decode before the first chunk plays; WAV is raw PCM and starts instantly. ~6× LAN bandwidth, but local network has plenty. ESPHome speaker-media-player docs explicitly recommend WAV at 16 kHz mono for fastest TTS startup.
- **`noise_suppression_level: 4` → `2`** ([line 482](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L482)). NS:4 runs WebRTC at max, adding per-frame latency before audio reaches HA's STT. Since `mic_aec` already cancels speaker echo, NS:2 is enough — keeps room-noise gate without the latency hit.
- **`pipeline_watchdog` `6s` → `12s`** ([line 779](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L779)). Watchdog runs between `on_stt_end` and `on_tts_start`, covering Gemini's "thinking" window. Gemini-flash typically responds in 2–5 s but can hit 6–10 s on cold cache. 6 s was killing slow responses; 12 s gives headroom without making real failures take forever.

**Server-side checklist (no YAML change, user verifies in HA):**
- HA → Settings → Voice Assistants → pipeline → "Streaming responses" enabled (Voice Chapter 11 introduced this; ~10× lower perceived latency on long replies).
- Confirm Gemini integration uses `gemini-2.0-flash`, not `gemini-pro` (latter is 3× slower for assistant-style replies).
- If you have local intents (lights, scenes, time), set the pipeline's "Prefer handling commands locally" toggle so simple commands skip Gemini entirely.

---

## Fix 7 — Music crashes "directly" on play (added 2026-04-29)

Symptom: starting music sometimes reboots the device immediately. "Directly" + immediate failure points at memory pressure / PSRAM transaction error during the burst of MP3-decode + resample + I2S that all hit memory simultaneously on the first audio frame.

- **`psram: speed: 200MHz` → `100MHz`** ([line 234](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L234)). Waveshare's reference configs ship at 200 MHz, but ESP-IDF's own docs warn 200 MHz PSRAM collides with the MPLL-sharing peripherals on ESP32-P4 ([esp-idf #17855](https://github.com/espressif/esp-idf/issues/17855)). 100 MHz is the documented middle option (ESPHome's `SPIRAM_SPEEDS` allow only `20 / 100 / 200` for ESP32-P4 — `120` is rejected by the validator). 100 MHz halves the chance of a PSRAM transaction error under burst load with negligible audio impact (16 kHz mono mix is well under the bandwidth budget).
- **`buffer_size: 512000` → `256000`** ([line 340](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L340)). The speaker media player allocates the full buffer up-front; 512 KB plus the announcement pipeline plus AEC ref + mixer + decoder buffers can OOM on the first MP3 frame when PSRAM is also under stress. 256 KB still gives a few seconds of audio runway and keeps memory pressure off the critical path.
- **(Already in place)** `mic_aec` for wake word + `safe_wake_word_stop` removed from `on_play` (Fix 2). This adds wake-word CPU during music; if Fix 7's two changes don't fully stop crashes, this is the next thing to roll back (revert Fix 2C).

**If music still crashes after the above:**
1. Drop `media_pipeline` from MP3 to FLAC (HA will re-encode upstream): FLAC decoding uses ~½ the CPU of MP3.
2. Match sample rates to skip the resampler: `media_pipeline.sample_rate: 16000` and `num_channels: 1` (audio quality drops, but eliminates the resampler entirely from the music path).
3. Roll back PSRAM to 200 MHz only AFTER confirming buffer_size + Fix 2 rollback didn't help, since 200 MHz is the most likely root cause.

---

## Fix 8 — Real root cause: SDIO Wi-Fi RX OOM in DRAM (added 2026-04-29)

After Fix 7 the device still crashed on music with this stack trace:

```
assert failed: sdio_process_rx_task sdio_drv.c:1260 (copy_payload)
MEPC: 0x4ff0794e   MCAUSE: 0x00000002 (illegal instruction = halted)
```

**What that line actually checks** (verified against the user's installed
`espressif__esp_hosted` 2.12.1, `host/drivers/transport/sdio/sdio_drv.c:1259-1260`):

```c
uint8_t * copy_payload = (uint8_t *)g_h.funcs->_h_malloc(buf_handle->payload_len);
assert(copy_payload);   // ← this fires when malloc returns NULL
```

The SDIO Wi-Fi bridge (ESP32-P4 ↔ ESP32-C6 over SDIO) gets a Wi-Fi packet,
tries to `malloc` a copy buffer, the malloc returns NULL, and the assert
halts the chip. `_h_malloc` is `heap_caps_malloc(..., MALLOC_CAP_DEFAULT)`,
which prefers **internal DRAM**. When music starts, the audio task stacks
+ MP3 decoder + AEC + mixer all grab DRAM, leaving nothing for the next
incoming Wi-Fi packet. Hence the crash is *correlated* with music start
but the actual failure is in the Wi-Fi RX path, not the audio path. PSRAM
speed/size and `buffer_size` (Fix 7) help only marginally — they don't
address the DRAM exhaustion.

**Fix:** move the audio task stacks into PSRAM with `task_stack_in_psram: true`
on every speaker-pipeline component that supports it. Each task stack is
4–8 KB; freeing 30–50 KB total in DRAM is plenty for the SDIO RX path.

```yaml
media_player:
  - platform: speaker
    id: board_player
    ...
    task_stack_in_psram: true   # ← NEW

speaker:
  - platform: mixer
    id: audio_mixer
    ...
    task_stack_in_psram: true   # ← NEW

  - platform: resampler
    id: music_resampler
    ...
    task_stack_in_psram: true   # ← NEW
```

The `i2s_audio_duplex` speaker (`hw_speaker`) does **not** accept this flag —
its task is owned by the duplex driver, not by the ESPHome speaker component.

**Why Fix 7's diagnosis was wrong:** PSRAM speed and `buffer_size` look
attractive when crashes correlate with music, but the actual crash trace
points unambiguously into the SDIO driver. The audio buffers themselves
*do* live in PSRAM by default; what doesn't is the FreeRTOS task stacks.
PSRAM 100 MHz / `buffer_size` reduction in Fix 7 are kept (they don't hurt),
but the workplan now treats Fix 8 as the actual fix.

**(User-side note — buffer_size 32768):** between Fix 7 and Fix 8 the user
manually dropped `buffer_size` from 256000 → 32768 while debugging. 32 KB is
above the documented minimum (4000) and works for low-bitrate music, but
gives only ~1 s of audio runway — risky on flaky Wi-Fi. With Fix 8 in place
the DRAM pressure is gone, so this can safely return to 256000 (or even back
to the 512000 default). Try `buffer_size: 256000` after confirming Fix 8
ends the SDIO crashes.

---

## Fix 11 — Full VA regression audit: wake word, STT, component migration (added 2026-04-29)

After the accumulated fixes, the VA was unreliable: wake word triggering inconsistently, failing with errors. Root cause was three mistakes in earlier fixes — identified by searching online and reading the intercom-api component source.

### 11a — Wake word microphone was wrong (Fix 2A regression)

**Fix 2A** changed `micro_wake_word.microphone` from `mic_raw` → `mic_aec`. This broke detection reliability. Confirmed across: ESPHome micro_wake_word docs, the intercom-api repo examples, and the component's own source comment (`pre_aec=true: raw mic for wake word detection` / `pre_aec=false: AEC-processed mic for voice assistant STT`).

AEC post-processes audio to cancel speaker echo — but it also distorts the phoneme frequency characteristics that the wake word model was trained on. `mic_raw` is the correct input for the model; `mic_aec` is only correct for the downstream STT.

**Reverted:**
- `microphone: mic_aec` → `microphone: mic_raw`
- `gain_factor: 8` → `gain_factor: 6`
- `probability_cutoff: 0.42` → `0.38` (both models)
- VAD `probability_cutoff: 0.42` → `0.40`

### 11b — `on_play` lost the wake word stop (Fix 2C regression)

Fix 2C removed `safe_wake_word_stop` from `media_player.on_play` on the assumption that `mic_aec` would suppress music spill. With `mic_raw` restored, this protection is needed again — `mic_raw` (pre-AEC) will pick up speaker output and false-trigger the wake word model.

**Restored:** `script.stop: restart_wakeword_safely` + `script.execute: safe_wake_word_stop` in `on_play`.

The user's pause→resume feature (Fix 5) still works: on_wake_word_detected pauses music, VA runs, and on_end resumes if no command was given.

### 11c — intercom-api external component API migration

The external component (`github://n-IA-hane/intercom-api`) was updated to a new version (`22dc384f`) that changed the `i2s_audio_duplex` schema:

| Old API | New API |
|---|---|
| `aec_id: aec_processor` | Removed — AEC auto-linked from `esp_aec` presence |
| *(didn't exist)* | `fir_decimator: custom` — **ESP32-P4 RISC-V fix**: the default SIMD decimator (`dsps_fird_s16`) is unreliable on the P4's RISC-V core (esp-dsp #117/#102); must use scalar float path |
| *(didn't exist)* | `buffers_in_psram: true` — moves non-DMA audio buffers to PSRAM (~15 KB internal RAM freed, further reduces SDIO RX OOM risk) |
| `aec_reference_buffer_ms: 100` | `aec_reference: ring_buffer` + `aec_reference_buffer_ms: 100` — ring_buffer mode gives better AEC delay alignment than the default `previous_frame` mode |
| `components: [i2s_audio_duplex, esp_aec]` | + `audio_processor` — `esp_aec` now `AUTO_LOAD`s `audio_processor` from the same external repo; must be listed explicitly |

All three are in the YAML now. `fir_decimator: custom` in particular should improve stability on the P4 since the SIMD decimator path was documented as unreliable on RISC-V cores.

---

## Fix 9 — SDIO streaming-mode 1MB DMA-aligned alloc fail (added 2026-04-29)

After Fix 8, the SDIO crash MOVED to a different code path:

```
assert failed: sdio_rx_get_buffer sdio_drv.c:896 (*buf)
register S9: 0x00100000 (= 1 MB)
```

This is the **streaming-mode** RX path (default in esp_hosted 2.12.1). It maintains one DMA-aligned buffer that grows to fit the largest seen packet. During music streaming, that buffer grew to 1 MB. `_h_malloc_align(1MB, 64)` requires a contiguous DMA-capable internal-RAM block — DMA allocations cannot live in PSRAM. 1 MB contiguous internal RAM does not exist on this device after the rest of the firmware loads. So the alloc fails → assert.

**Fix:** switch from streaming RX to per-packet RX via sdkconfig:

```yaml
esp32:
  framework:
    type: esp-idf
    advanced:
      sdkconfig_options:
        CONFIG_ESP_HOSTED_SDIO_OPTIMIZATION_RX_STREAMING_MODE: n
        CONFIG_ESP_HOSTED_SDIO_OPTIMIZATION_RX_NONE: y
```

Per-packet mode bounds each malloc to actual packet size (~1.5 KB for typical Wi-Fi MTU). The Kconfig comment for RX_NONE says *"Use this if unsure"* — it's the safe fallback path. There's a small SDIO throughput cost (extra transactions per packet vs. streaming a queued batch), but at music-streaming bitrates (~32 KB/s peak) the overhead is negligible.

**(Caveat)** If Fix 9 is found to degrade STT or anything else, swap RX_NONE for RX_MAX_SIZE (`CONFIG_ESP_HOSTED_SDIO_OPTIMIZATION_RX_MAX_SIZE: y`). MAX_SIZE is also per-packet but always reads max size to skip the size-query SDIO transaction — slightly faster than NONE, still bounded.

---

## Fix 10 — STT regression: `stt-stream-failed` after Fix 6 (added 2026-04-29)

**Symptom:** after flashing Fix 6, the device wake-words and listens, but HA returns:
```
Error: stt-stream-failed - speech-to-text failed
```

Source confirmed at `homeassistant/components/assist_pipeline/pipeline.py:975` — STT engine returned a non-SUCCESS result (audio reached the engine but transcription failed).

**Root cause:** Fix 6's `noise_suppression_level: 4 → 2` broke a deliberately balanced trio of mic-side knobs that the prior author had tuned together (per the original comment block in the YAML):

| Setting | Original | After Fix 6 | Effect of the change |
|---|---|---|---|
| `noise_suppression_level` | 4 (max) | 2 (medium) | Less aggressive WebRTC NS |
| `auto_gain` | 31 dBFS (max) | 31 dBFS (unchanged) | AGC sees more residual noise → over-amplifies |
| `volume_multiplier` | 4.0 | 4.0 (unchanged) | Stacked 4× pre-amp now clips the over-amplified signal |
| `mic_gain` (codec) | 24 dB | 24 dB (unchanged) | Already-loud codec output |

The combo NS:4 + AGC:31dBFS + ×4 multiplier was explicitly noted as "no distortion." Dropping NS to 2 lets more residual signal through, the AGC compensates upward, and the 4× post-multiplier then clips. **Clipped audio is noise to the STT engine** — which returns `SpeechResultState.ERROR` rather than empty text — hence `stt-stream-failed` instead of the gentler `stt-no-text-recognized`.

**Fix:** revert just the NS knob:

```yaml
voice_assistant:
  ...
  noise_suppression_level: 4    # was 2 (Fix 6) — reverted
  auto_gain: 31dBFS
  volume_multiplier: 4.0
```

The other Fix 6 changes (`announcement_pipeline.format: WAV`, `pipeline_watchdog: 12s`) only affect TTS *output* and the Gemini "thinking" timeout — neither touches the STT input path. They stay.

**Fix 9 was reverted in Fix 12** — streaming mode restored (see below). STT should now work without that blocker.

**If STT *still* fails** (other diagnostics in priority order):
2. **Roll back Fix 2A** (wake word back to mic_raw) — separates the two consumers onto different mic buffers, removes any AEC reference timing skew that two consumers might cause.
3. **Bump `aec_reference_buffer_ms: 100 → 150`** ([line 198](esphome-config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml#L198)) — if the speaker pipeline got slower after Fix 8 (`task_stack_in_psram` + PSRAM 100 MHz), the 100 ms AEC reference might over-subtract.
4. **Drop `volume_multiplier: 4.0 → 2.0`** — defensive; halves post-AGC gain to leave headroom against any residual AGC overshoot.

Try #1–4 in order. Each should be verified independently (single change at a time).

---

## Fix 12 — SDIO streaming mode restored + animation cooldown (added 2026-04-30)

### 12a — SDIO Fix 9 reverted: streaming mode restored

Reading `sdio_reg.h:41` confirmed: `ESP_RX_BUFFER_SIZE = 1536` is hardcoded, and the size check only applies when `H_SDIO_HOST_RX_MODE != H_SDIO_HOST_STREAMING_MODE` (`sdio_drv.c:427`). The C6 slave is compiled in streaming mode and sends batched SDIO transfers. A 2869-byte batch (two bundled TCP frames) exceeds 1536 → per-packet modes always reject it → TCP abort → announcement download fails → `speaker_media_player.pipeline:117: ESP_FAIL` → media player goes IDLE mid-speech → clock screen cuts in.

**Root conflict:** per-packet host modes and streaming slave are fundamentally incompatible. STREAMING_MODE must be used on the host.

The Fix 9 SDIO sdkconfig overrides are removed. The streaming mode's 1MB crash (`sdio_drv.c:896`) was the slave's 20-bit length register (`ESP_SLAVE_LEN_MASK = 0xFFFFF`) hitting its maximum — a burst or roll-over event. With the DRAM headroom from Fix 8 (`task_stack_in_psram`) and Fix 11c (`buffers_in_psram`), the system has more internal RAM available and the burst event is less likely to exhaust it.

### 12b — Speaking animation no longer cut short by clock screen

Two complementary changes:

**`voice_assistant.on_end` — 1.5 s animation cooldown before returning to clock:**

```yaml
else:
  - delay: 1500ms      # NEW — let Ivy finish speaking naturally
  - script.execute:
      id: set_anim_state
      target_state: 0
```

**`media_player.on_idle` — guard: only return to clock from state 4 (Now Playing):**

```yaml
# Before: !id(va_speaking) — transitions to state 0 from any state
# After: only from state 4 (music was playing, now idle)
- if:
    condition:
      lambda: 'return !id(va_speaking) && id(current_anim_state) == 4;'
    then:
      - script.execute: { id: set_anim_state, target_state: 0 }
```

The guard prevents the media-player IDLE event (which fires when the TTS announcement channel goes idle) from cutting the state-3 animation during the 1.5 s cooldown window. Without it, the announcement pipeline going IDLE would immediately override state 3 → 0 — exactly the "clock cuts the speaking animation" symptom. With it, `on_idle` only transitions to state 0 when coming from state 4 (Now Playing after music stops), which is the only legitimate state 0 transition via `on_idle`.

---

## Verification (test on hardware after flash)

Test in this order — each builds on the previous:

1. **Weather** — Reboot, wait ~5 s for HA to push weather. Both temperature *and* icon should show. If icon shows but temp says `-- °C`, the entity_id fix didn't take.
2. **Logs** — Tail `esphome logs --no-states /config/waveshare-p4-touch-lcd-4c-ai-ivy.yaml`. Idle for 30 s; you should see < 5 lines/min. Trigger one wake-word cycle; you should see only diagnostic lines (Wake word detected, STT heard, TTS text). Without `--no-states` you'll still see `[S][media_player]:` from aioesphomeapi (see Fix 4 callout).
3. **Wake-word reliability while idle** — Say wake word 10× over 5 minutes from normal speaking distance. Should detect ≥ 8/10. If <6/10, drop probability_cutoff to 0.40 then 0.38.
4. **Wake-word during music** — Start a music stream from HA. Say wake word. Music should *pause* (not stop), VA should listen.
5. **Pause/resume timeout** — Repeat #4 but stay silent. After ~10 s VA times out → music resumes automatically and Now Playing screen comes back.
6. **Pause + command** — Repeat #4, but issue a real command ("what time is it?"). VA replies, music stays paused, screen returns to clock.
7. **Speed perception (Fix 6)** — Time from end of user speech to first TTS audio chunk should drop noticeably (WAV transcode is faster than FLAC). Long replies (multi-sentence) should no longer get cut off mid-sentence (12 s watchdog).
8. **Music stability soak (Fix 7 + 8)** — Start music 10× in a row from HA, wait for first chunk each time. No reboots. Then play continuously for 20 minutes. If reboots persist with `assert failed: sdio_process_rx_task`, the DRAM pressure is still too high — see Fix 8 for next steps. After Fix 8 lands, also try restoring `buffer_size: 256000` for smoother playback.

---

## Rollback table (if any change breaks something)

| Symptom | Revert |
|---|---|
| Wake word doesn't trigger at all after change | **Already fixed in Fix 11** — was incorrectly on mic_aec; now mic_raw with gain_factor=6 and cutoffs 0.38/0.40 |
| SDIO crashes / device reboots during music + wake | Revert `safe_wake_word_stop` to `on_play` (Fix 2C) — **already re-added in Fix 11** |
| Boot hangs after wake word | Drain 600ms → 900ms (Fix 3) |
| Music doesn't resume on timeout | Swap `media_player.play` → `media_player.pause_resume` (Fix 5) |
| Weather still missing | Verify entity exists in HA Developer Tools → States, then check API encryption key matches |
| TTS audio crackles or stutters after WAV switch | Revert `announcement_pipeline.format: WAV` → `FLAC` (Fix 6) |
| `stt-stream-failed` after Fix 6 | Revert `noise_suppression_level` 2 → 4 (Fix 10). NS:2 broke the NS+AGC+multiplier balance → clipped audio. |
| Background noise/music-spill garbles STT after NS drop | After reverting NS to 4, if you still want to chase latency, try `noise_suppression_level: 3` and `volume_multiplier: 3.0` together (keep balance). |
| Long Gemini reply still cuts off | Bump `pipeline_watchdog` delay 12s → 20s (Fix 6) |
| Music underruns / stutters (not crashes) after buffer drop | Bump `buffer_size` 256000 → 384000 (Fix 7) |
| Crashes persist on music start | See Fix 7 fallback list (FLAC media_pipeline → mono 16k → revert PSRAM) |
| `assert failed: sdio_process_rx_task` / `copy_payload` reboot | DRAM exhaustion in SDIO RX path. Apply Fix 8 (`task_stack_in_psram: true` on speaker_media_player + mixer + resampler). Fix 7 alone won't help — root cause is task stacks in DRAM, not audio buffers in PSRAM. |
| Music underruns after task_stack_in_psram (Fix 8) | Bump `buffer_size` 32768 → 256000 (Fix 7 was over-tightened in user debugging session) |
