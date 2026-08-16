# ESPHome LVGL UI Reference

## Display Setup

### MIPI DSI (ESP32-P4)

```yaml
display:
  - platform: mipi_dsi
    id: my_display
    model: WAVESHARE-ESP32-P4-WIFI6-TOUCH-LCD-3.4C
    transform:
      mirror_x: true   # for Pepper's Ghost hologram
```

### LVGL Configuration

```yaml
lvgl:
  rotation: 180           # uses P4 PPA hardware accelerator
  displays: [my_display]
  buffer_size: 10%         # lower with PSRAM
  byte_order: little_endian
  bg_color: 0x000000
```

**Important**: Rotation should be in the `lvgl:` block, not the `display:` block,
to use hardware-accelerated rotation on P4.

## Widget Reference

### Label

```yaml
- label:
    id: my_label
    text: "Hello"
    align: CENTER
    y: -50
    text_color: 0xFFFFFF
    text_font: roboto_48
    long_mode: SCROLL_CIRCULAR  # for scrolling text
```

**Dynamic update**:
```yaml
- lvgl.label.update:
    id: my_label
    text: !lambda 'return std::string("Updated");'
```

### AnimImg (Animated Image)

```yaml
- animimg:
    id: state_anim
    align: CENTER
    src: [frame_01, frame_03, frame_05]
    duration: 2600ms
    repeat_count: forever
    auto_start: false
    hidden: true
```

**Switch animation source at runtime**:
```yaml
- lvgl.animimg.update:
    id: state_anim
    src: [new_frame_01, new_frame_03]
    duration: 2000ms
    repeat_count: forever
    auto_start: true
```

### Obj (Container)

Used as screen regions for mutual exclusion:

```yaml
- obj:
    id: screen_a
    width: 100%
    height: 100%
    bg_color: 0x000000
    border_width: 0
    hidden: false
    widgets: [...]

- obj:
    id: screen_b
    width: 100%
    height: 100%
    hidden: true
    widgets: [...]
```

**Toggle screens**:
```yaml
- lvgl.widget.update:
    id: screen_a
    hidden: true
- lvgl.widget.update:
    id: screen_b
    hidden: false
```

### Spinner

```yaml
- spinner:
    id: boot_spinner
    width: 80
    height: 80
    align: CENTER
    spin_time: 1s
    arc_length: 60deg
```

### Processing Overlay Pattern

Transparent overlay on top of animation:

```yaml
- obj:
    id: processing_screen
    width: 100%
    height: 100%
    bg_opa: TRANSP        # lets animation show through
    border_width: 0
    hidden: true
    widgets:
      - label:
          id: processing_label
          text: "Processing..."
          align: BOTTOM_MID
          y: -40
```

## Screen State Machine

### Pattern

```yaml
globals:
  - id: current_anim_state
    type: int
    restore_value: false
    initial_value: '0'

script:
  - id: set_anim_state
    mode: restart
    parameters:
      target_state: int
    then:
      - if:
          condition:
            lambda: 'return id(current_anim_state) != target_state;'
          then:
            - globals.set:
                id: current_anim_state
                value: !lambda 'return target_state;'
            # Hide all screens, show target
            - ...
```

### States Used in This Repo

| State | Name | Screens Visible |
|---|---|---|
| 0 | Idle | clock_weather_screen |
| 1 | Listening | state_anim (listening frames) |
| 2 | Processing | state_anim (processing frames) + processing_screen overlay |
| 3 | Replying | state_anim (replying frames) |
| 4 | Now Playing | now_playing_screen |

## Image Configuration

### Defaults Block

```yaml
image:
  defaults:
    type: rgb565
    byte_order: little_endian
    transparency: alpha_channel
    resize: 240x240
```

### Loading Images

```yaml
  images:
    - file: "ha-friday/Assets/listening/listening-1.png"
      id: listening_01
```

**Strategy**: Use every-other-frame (1, 3, 5, 9, 13) for smooth animation
with half the flash cost.

## Font Configuration

### Google Fonts

```yaml
font:
  - file: "gfonts://Roboto"
    id: roboto_48
    size: 48
```

### MDI Weather Icons

```yaml
  - file: 'https://github.com/Templarian/MaterialDesign-Webfont/raw/master/fonts/materialdesignicons-webfont.ttf'
    id: mdi_weather_font
    size: 96
    glyphs:
      - "\U000F0597"  # weather-sunny
      - "\U000F0590"  # weather-cloudy
      # ... etc
```

### Swedish Character Support

Include explicit glyph ranges for ÅÄÖåäö when using text that may contain
Swedish characters:

```yaml
    glyphs:
      - " "
      - "'"
      - '!"#$%&()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~°ÅÄÖåäö'
```

## Backlight Control

```yaml
output:
  - platform: ledc
    pin: GPIO26
    id: display_backlight_pwm
    frequency: 1000Hz
    inverted: true

light:
  - platform: monochromatic
    output: display_backlight_pwm
    name: "Screen Brightness"
    id: screen_backlight
    restore_mode: ALWAYS_ON
```

### Idle Timer Pattern

```yaml
script:
  - id: screen_idle_timer
    mode: restart          # resets on each new activity
    then:
      - delay: 5min
      - light.turn_off: screen_backlight
```
