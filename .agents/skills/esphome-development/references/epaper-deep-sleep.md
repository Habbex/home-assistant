# ESPHome E-Paper & Deep Sleep Reference

## Deep Sleep Architecture

E-paper devices use a "wake → work → sleep" cycle to maximize battery life.
The display retains its image with zero power, so the device only needs to be
awake during data refresh.

### Lifecycle

```
ESP32 wakes (timer or GPIO interrupt)
  → Measure battery (enable ADC → read → disable)
  → Connect WiFi (with timeout)
  → Connect HA API (with timeout)
  → Wait for data sync (5s)
  → Render display (component.update)
  → Delay for physical refresh (~25-30s for Spectra 6)
  → Enter deep sleep
```

### Configuration

```yaml
deep_sleep:
  id: deep_sleep_control
  run_duration: 90s        # watchdog: max awake time
  sleep_duration: 6h       # sleep between refreshes
  wakeup_pin:
    number: GPIO2          # manual wake button
    inverted: true
    mode:
      input: true
      pullup: true
```

### Manual Deep Sleep Entry

```yaml
- deep_sleep.enter:
    id: deep_sleep_control
```

## Battery Monitoring

### ADC with Switched Divider

To prevent the voltage divider from draining the battery during sleep, use a
GPIO-switched circuit:

```yaml
switch:
  - platform: gpio
    pin: GPIO6
    id: battery_adc_enable
    restore_mode: ALWAYS_OFF

sensor:
  - platform: adc
    pin: GPIO1
    id: battery_voltage
    update_interval: never    # manual trigger only
    attenuation: 11db
    filters:
      - multiply: 2.0        # voltage divider ratio
```

### Measurement Sequence

```yaml
on_boot:
  - switch.turn_on: battery_adc_enable
  - delay: 100ms
  - component.update: battery_voltage
  - delay: 50ms
  - switch.turn_off: battery_adc_enable
```

### Battery Percentage Calculation

```yaml
sensor:
  - platform: copy
    source_id: battery_voltage
    name: "Battery Level"
    unit_of_measurement: "%"
    filters:
      - calibrate_linear:
          - 3.2 -> 0      # empty LiPo
          - 4.2 -> 100     # full LiPo
      - clamp:
          min_value: 0
          max_value: 100
```

### Charging Detection

Simple threshold-based detection (hardware mod recommended for precision):

```yaml
binary_sensor:
  - platform: template
    name: "Battery Charging"
    lambda: |-
      if (!id(battery_voltage).has_state()) return false;
      return id(battery_voltage).state > 4.25f;
```

## Spectra 6 E-Paper Display

### Custom Component Setup

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/acegallagher/esphome-bigink
      ref: main
      path: bigink_component
    components: [seeed_epaper_spectra6]
```

### Pin Configuration (EE02 Driver Board)

```yaml
display:
  - platform: seeed_epaper_spectra6
    id: epaper_display
    update_interval: never
    clk_pin: 7
    mosi_pin: 9
    cs_master_pin: GPIO44
    cs_slave_pin: GPIO41
    dc_pin: GPIO10
    reset_pin: GPIO38
    busy_pin:
      number: GPIO4
      inverted: true
      mode: {input: true, pullup: true}
    power_pin: GPIO43
    flip_mode: false
```

### Rendering in Lambda

```cpp
lambda: |-
  // Color constants
  auto BLACK  = Color(0, 0, 0);
  auto WHITE  = Color(255, 255, 255);
  auto RED    = Color(255, 0, 0);
  auto GREEN  = Color(0, 255, 0);
  auto BLUE   = Color(0, 0, 255);
  auto YELLOW = Color(255, 255, 0);

  // Layout
  const int W = 1600;
  const int H = 1200;
  const int MARGIN = 20;

  // Clear
  it.fill(WHITE);

  // Draw text
  it.printf(x, y, id(font_id), color, TextAlign::CENTER, "text");

  // Draw shapes
  it.filled_rectangle(x, y, w, h, color);
  it.rectangle(x, y, w, h, color);        // outline only
  it.line(x1, y1, x2, y2, color);
  it.draw_pixel_at(x, y, color);          // dotted lines
```

### Text Alignment Options

```
TOP_LEFT, TOP_CENTER, TOP_RIGHT
CENTER_LEFT, CENTER, CENTER_RIGHT
BOTTOM_LEFT, BOTTOM_CENTER, BOTTOM_RIGHT
```

## Power Optimization Checklist

- [ ] `fast_connect: true` in WiFi config
- [ ] No captive portal / fallback AP (drains battery)
- [ ] ADC divider switched off during sleep
- [ ] `update_interval: never` on display
- [ ] `run_duration` set as watchdog
- [ ] WiFi and API connection timeouts configured
- [ ] Sufficient delay between `component.update` and `deep_sleep.enter`
- [ ] Consider static IP for faster connection
- [ ] Consider reduced WiFi TX power if near router

## Calendar Data Format

Events are passed from HA via `input_text` helpers in pipe-delimited format:

```
TITLE|START_DAY|END_DAY|START_HOUR|END_HOUR|ALLDAY
```

| Field | Description | Values |
|---|---|---|
| TITLE | Event name | UTF-8 string |
| START_DAY | Day index | 0=Mon, 1=Tue, ..., 6=Sun |
| END_DAY | Day index | Same as START_DAY |
| START_HOUR | Decimal hour | 9.5 = 09:30, 13.75 = 13:45 |
| END_HOUR | Decimal hour | Same format |
| ALLDAY | All-day flag | 1 = all-day, 0 = timed |

Weather forecast is semicolon-separated:
```
Day|condition|high|low;Day|condition|high|low
```
