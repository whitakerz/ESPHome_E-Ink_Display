
# Home Assistant E-Ink Dashboard

A simple, battery-powered, Waveshare 2.9" e-ink display for Home Assistant, powered by ESPHome & ESP32.

## Project Overview

This project is a low-power, always-ready e-ink display for Home Assistant that runs on a Lolin D32 (ESP32) microcontroller. 
It was designed to display indoor and outdoor temperature, humidity, and other environmental readings on a battery-powered device.


**Key features:**
- Battery and 5 V input voltage measurement using ADC (through a [resistor network](https://github.com/whitakerz/Battery-ESP32/blob/main/Resistor%20Network.txt)).
- Battery percentage estimation with nonlinear calibration
- Automatic day/night deep sleep control
- Home Assistant event logging for voltage and sleep cycles
- OTA updates and encrypted API communication
- Because existing code and enclosures did not meet my requirements, I combined multiple ESPHome projects, wrote custom configuration, 
and had a custom case designed on Fiverr.


## Hardware Used

- **Display:** Waveshare 2.9" E-Ink Display (V2, R2), 296x128 px, partial refresh, SPI interface
- **Microcontroller:** Lolin D32 (ESP32-based)
- **Battery:** Single-cell LiPo, monitored via ADC
- **Case:** Custom-designed and printed via Fiverr
- **Resistors:** Voltage divider networks for battery and 5V sensing
- **Miscellaneous:** 200Ω resistor for onboard LED, JST connectors, wiring

## Wiring Diagram

```
Power & Dividers (left)                             ESP32 (center)             E-Ink Display (right)
───────────────────────────                   ────────────────────────      ─────────────────────────────
          +5V supply                            ┌──────────────────┐            Waveshare 2.9" v2-r2
              |                                 │      ESP32       │              (3.3V logic)
           [20kΩ]                               │                  │
              |                                 │  GPIO18 ──────────────── CLK  ─────────────► SCK
          ──● Node A (GPIO36 sense) ────────────┤◄─ GPIO36 (ADC 5V)|        (SPI clock)
              |                                 │                  |
              |                                 │  GPIO23 ──────────────── MOSI ─────────────► DIN
              |                                 │                  |    (SPI data out)
              |                                 │                  |
            (series link)                       │  GPIO15 ───────────────── CS   ─────────────► CS
              |                                 │                  |
          ──● Node B (GPIO39 wake) ─────────────┤◄─ GPIO39 (wakeup)|
              |                                 │                  |
           [30kΩ]                               │  GPIO17 ───────────────── DC   ─────────────► D/C
              |                                 │                  |
            GND                                 │  GPIO16 ───────────────── RST  ─────────────► RST
                                                │                  |
      4.2V (LiPo)                               │  GPIO4  ◄──────────────── BUSY ─────────────◄ BUSY
              |                                 │                  |
           [36kΩ]                               │  3V3  ─────────────────── VCC  ─────────────► VCC (3.3V)
              |                                 │  GND  ─────────────────── GND  ─────────────► GND
          ──● Node C (GPIO32 batt) ─────────────┤◄─ GPIO32 (ADC batt)
              |                                 │                  |
           [100kΩ]                              │  GPIO22 ──► LED (Awake) ──►───┐
              |                                 │                  |       (LED)|
            GND                                 └──────────────────┘            └───[200Ω] GND

Legend:
  [value]   = resistor
  ● Node A  = divider tap for 5V sense (GPIO36)
  ● Node B  = divider mid / wake input (GPIO39)
  ● Node C  = divider tap for LiPo sense (GPIO32)
  LED       = “Awake LED” driven by GPIO22 (add a series resistor if needed)
```

GPIO39 doubles as a wake pin for deep sleep. GPIO22 drives an LED to indicate awake state. Both battery and 5V input are monitored via resistor dividers.

## Power Management

The display is battery-powered and relies on ESPHome's deep sleep functionality:
- Runs for ~1 minute per wake cycle (configurable)
- Daytime sleep of ~15 minutes between updates
- Night mode enters 8-hour deep sleep after 10 PM
- ESP32 is underclocked to 80 MHz for power savings

## Display Layout

The panel is 296 x 128 px (rotated 90°) and is laid out as two single-line rows:
outdoor on top, indoor below. Each row carries its own column labels in a 13 pt
font above the values.

```
      0                                                              296
      ┌────────────────────────────────────────────────────────────────┐
    0 │              Temp          Humidity       Soil                 │
      │  ╔══════╗                                                      │
   13 │  ║ tree ║    92°           51%            39%                  │
      │  ╚══════╝                                                      │
   54 │              Temp          Humidity       CO2                  │
      │  ╔══════╗                                                      │
   67 │  ║ home ║    74°           60%            565                  │
      │  ╚══════╝                                                      │
      │                                                                │
  110 │  08/09/2026 12:46:44 PM                    +5V   ▛▀▀▀▀▜        │
  128 └────────────────────────────────────────────────────────────────┘
         x=10    x=50           x=160          x=225      x=220  x=250
       IMAGE_    TEMP_          HUMIDITY_      RIGHT_COL_
       HORIZONTAL HORIZONTAL    HORIZONTAL     HORIZONTAL
```

- Top row (tree icon) shows outdoor temperature, humidity, and planter soil moisture.
- Bottom row (house icon) shows printer-enclosure temperature, humidity, and office CO2.
- Bottom-left shows the timestamp of the last update.
- Bottom-right contains the battery indicator and the `+5V` charging marker.

### Entity map

| Column | ESPHome id | Home Assistant entity | Units |
|---|---|---|---|
| Outdoor temp | `temperature_outside` | `sensor.calibrated_porch_temperature` | °F |
| Outdoor humidity | `humidity_outside` | `sensor.indoor_outdoor_meter_0a71_humidity` | % |
| Outdoor right col | `planter_moisture` | `sensor.calibrated_planter_capacitance_moisture` | % |
| Indoor temp | `temperature_printer` | `sensor.printer_barometer_temperature` | °F |
| Indoor humidity | `humidity_printer` | `sensor.printer_barometer_humidity` | % |
| Indoor right col | `co2_office` | `sensor.co2_meter_carbon_dioxide` | ppm |
| Clock | `home_time` | HA `time` platform | — |

All displayed values are pulled from Home Assistant. The node itself only measures
its own battery/5V rails and WiFi signal.

## Software (ESPHome)

The configuration uses ESPHome to pull data directly from Home Assistant entities. Battery voltage is displayed using a nonlinear LiPo discharge curve. 
The code includes logic for day/night cycles, custom refresh rates, and battery percentage calculations.

### Configuration files

| File | Status |
|---|---|
| `Battery_08_09_26.yaml` | **Current.** Flash this one. |
| `Battery_10_04_25.yaml` | Previous baseline, kept for rollback. |

### Changes in `Battery_08_09_26.yaml`

Both barometric-pressure readouts were dropped and their slots reused. The layout,
coordinates, fonts, and draw order are otherwise untouched.

| Slot | Was | Now |
|---|---|---|
| Outdoor row, right column | `sensor.forecast_pressure_mmhg` | Planter soil moisture |
| Indoor row, right column | `sensor.printer_barometer_pressure` | Office CO2 (ppm) |

The complete diff against `Battery_10_04_25.yaml` is five hunks:

1. `sensor:` — `forecast_pressure` → `planter_moisture`
2. `sensor:` — `pressure_printer` → `co2_office`
3. lambda — constant renamed `PRESURE_HORIZONTAL` → `RIGHT_COL_HORIZONTAL`
   (same value, `HUMIDITY_HORIZONTAL + 65`; cosmetic only)
4. lambda — outdoor right column reads `planter_moisture`, format `"%.0f%%"`
5. lambda — indoor right column reads `co2_office`, format `"%.0f"`

Both replacements use `id(my_font)` at 30 pt at identical coordinates, so the two
rows keep their original visual weight. Draw-call count is unchanged at 17.

Deep sleep, `on_boot`, `sensors_ready`, `voltage_present`, the `enter_sleep`
script, the 2-minute refresh interval, fonts, images, and the display hardware
settings are all byte-identical to the previous version.

### Notes on the current layout

- **`forecast_condition` is declared but never drawn.** The `text_sensor` for
  `weather.forecast_home` and `partly_cloudy.bmp` are both defined and consume no
  display space. This is long-standing and left as-is. The canvas has no free
  space above `y=102` — the two data rows and the 38 pt temperature glyphs consume
  it all — so a weather glyph would have to go in the footer beside the clock.
- **CO2 column width.** The right column starts at x=225, leaving 71 px to the
  panel edge. Three digits fit comfortably; a 4-digit reading (>=1000 ppm) needs
  roughly 58-66 px at 30 pt. It fits, but with little margin.
- **Night threshold vs. comment.** `night_sleep_time` is commented
  `# 1st sleep time after 10pm`, but the script tests `time.hour >= 21`, which is
  9 PM. One of the two is wrong; left alone as it is sleep behaviour, not display.

## Case

Since no suitable enclosure existed, I commissioned a custom design via Fiverr. The case supports the display, microcontroller, and battery while remaining easy to open for maintenance. STL files are included in the repository.

## Credits and Inspiration

- ESPHome Feature Requests #1109
- kotope/esphome_eink_dashboard
- hanspeda/esphome_homeassistant_display
- Plawasan’s E-Ink Gist

## Getting Started

1. Clone this repository:
   ```bash
   https://github.com/whitakerz/Battery-ESP32.git
   ```
2. Flash the ESPHome config:
   ```bash
   esphome run Battery_08_09_26.yaml
   ```
3. Connect the device to Home Assistant.
4. Print and assemble the custom case.
