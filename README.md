# Plant Humidity Monitor

A Home Assistant integration for real-time soil moisture monitoring across multiple plants using ESP32 and capacitive soil sensors.

> **Work in Progress** — v1 used resistive sensors and hit a known limitation with peat soil. This repo is being updated to use capacitive sensors (Cytron Maker Soil Moisture). See the [Sensor Upgrade](#-sensor-upgrade-in-progress) section.

**Read the full story:** [The Soil Sensor Saga](https://noawinged.me/entry/soil-sensor-saga) — a blog post about building this system, transition, and discovering that AI can give you beautifully confident wrong answers.

## Hardware Setup

### What You Need

- **ESP32-S3-DevKit-C-1** — the brain (~$10)
- **4x Soil Moisture Sensors** — see [Sensor Upgrade](#-sensor-upgrade-in-progress) for which ones to buy
- **Dupont Jumper Cables** — connectors (~$2)
- **Home Assistant** — automation and history

**Total cost: less than a plant pot.**

### Pin Configuration

The ESP32-S3 has quirks. WiFi interferes with ADC2, PSRAM uses GPIO 35-37, and GPIO3 is a strapping pin. This config avoids all of those.

```
ADC1 Pins (WiFi-stable, for moisture readings):
  GPIO1 → Sensor 1
  GPIO2 → Sensor 2
  GPIO4 → Sensor 3
  GPIO5 → Sensor 4

GPIO Pins (for sensor power switches):
  GPIO38 → Power Sensor 1
  GPIO39 → Power Sensor 2
  GPIO40 → Power Sensor 3
  GPIO41 → Power Sensor 4
```

**Why power switches?** Sensors powered continuously cause electrolysis in soil. This setup powers each sensor for only 2 seconds during measurement — about 48 seconds per day instead of 24 hours continuously. This matters more for resistive sensors (exposed metal), but is good practice regardless.

## ⚠️ Sensor Upgrade In Progress

The original build used cheap **resistive** soil moisture sensors (FC-28 style, ~$5 for four). These work by passing electrical current between two exposed metal probes through the soil.

**The problem with resistive sensors in peat:**

Peat soil shrinks as it dries. As it contracts, it pulls away from the sensor probes, creating tiny air pockets. Since resistive sensors require direct galvanic contact with soil to measure anything, even a small air gap breaks the circuit completely — the sensor reads 0% regardless of actual moisture levels nearby. The only fix is to press the probe back into soil manually.

This turns "automated plant monitoring" into "walking around re-pressing sensors with your finger every few days." Not quite the dream.

**Why capacitive sensors are better for this:**

Capacitive sensors emit an electric field around the probe rather than passing current through it. That field can read through small gaps — the peat shrinkage still affects readings slightly, but doesn't cause complete circuit breaks. Isolated probes also don't corrode in acidic peat substrate over time.

**What's changing:** The configs in this repo are being updated for the **Cytron Maker Soil Moisture** capacitive sensor. Same ESP32 pins, same YAML structure, better physics. Calibration values will be updated when the hardware swap is complete.

## Installation

### 1. Flash the ESP32

```bash
pip3 install esphome
cd esp32/
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your WiFi credentials
esphome run plant_humidity_monitor-prod.yaml
```

Use `prod.yaml` (15-minute intervals) for daily use. The other two are for development:
- `debug.yaml` — real-time voltage for hardware testing
- `live.yaml` — 2-second refresh for calibration

**Tip:** Compile locally on your machine, not in Home Assistant. HA Green takes 10+ minutes per build; a laptop takes ~3 minutes. That difference changes how willing you are to iterate.

### 2. Add Home Assistant Configuration

**First:** Make sure your `~/.homeassistant/config/configuration.yaml` has packages enabled:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

**Then:** Copy the Home Assistant config package:

```bash
cp -r home_assistant/config/packages/plant_humidity_monitor \
  ~/.homeassistant/config/packages/
```

Restart Home Assistant. You should see four raw voltage sensors (`sensor.soil_moisture_1_raw`, etc.) appear in **Settings → Devices & Services**.

### 3. Calibration

The system sends **raw voltages** to Home Assistant. You define the math in software:

- **Air / Dry soil:** ~3.14V / ~2.10V
- **Water / Wet soil (just watered):** ~1.05V / ~1.15V
- **Linear mapping:** convert volts to percentage

For each plant:
1. Put sensor in dry soil from your pot. Record the voltage.
2. Water thoroughly. Record the voltage 15 minutes later.
3. Set these in the Home Assistant sliders (`Plant X Dry/Wet Limit`).

**Calibrate with actual cables in place.** Cable length affects voltage — a 2.5m cable reads differently than 0.5m. Always do final calibration with your real setup, not on the desk.

### 4. Dashboard

Copy the dashboard cards:

```bash
cp home_assistant/dashboard/*.yaml \
  ~/.homeassistant/config/lovelace/
```

Or configure manually in the HA UI. The key template sensors are `NS Plant 1/2/3/4` — they convert raw voltage to moisture percentage.

## Configuration Files

### ESP32 Configuration (`esp32/plant_humidity_monitor-prod.yaml`)

- **15-minute measurement cycle**
- **Power management script** — safely powers sensors on/off
- **Raw voltage output** — Home Assistant handles calibration logic

### Home Assistant Core (`home_assistant/config/packages/plant_humidity_monitor/core.yaml`)

**Template sensors** converting raw voltage to percentage:
- Per-plant calibration (dry/wet limits)
- Fertilizer offset slider for multi-day tracking
- Debug toggle to show raw voltage
- Icon switches to water-alert below individual thresholds

Thresholds: Plant 1 & 4 below 50%, Plant 2 below 60%, Plant 3 below 40%.

### Home Assistant User Config (`home_assistant/config/packages/plant_humidity_monitor/user_config.yaml`)

Sliders and buttons: debug toggle, fertilizer offset, calibration sliders per plant, save/restore profiles.

## Troubleshooting

### Sensors show moisture dropping impossibly fast

**Example:** You water (90%), and within hours the reading crashes to 0% while the soil is still obviously wet.

**Root cause:** This is the resistive sensor air gap problem described above. Peat shrinks as it dries, pulling away from the metal probes and breaking the electrical circuit. The sensor measures conductivity between its prongs — no contact means no reading.

**Immediate fix:** Press each sensor firmly back into soil with solid contact.

**Real fix:** Upgrade to capacitive sensors (see [Sensor Upgrade](#-sensor-upgrade-in-progress)).

### Sensor readings drift over time

Soil conditions change as it dries and re-wets. Use the calibration sliders in Home Assistant to adjust on the fly rather than reflashing firmware. That's why calibration lives in HA rather than the ESP32 config.

## Why This Design

### Raw voltage in firmware, calibration in HA

Calibration changes — soil type changes, you add a plant, you swap sensors. Doing the logic in HA means tweaking a slider instead of reflashing.

### 15-minute intervals with power management

Plugged in, not battery-powered, so no deep sleep. Power-gating the sensors prevents electrolysis. 15 minutes is long enough to see trends, short enough to catch problems before they're serious.

### Four plants, hardcoded

This started as a one-off for four ferns. If you need more, duplicate the sensor blocks in the YAML and template sensor blocks in `core.yaml`.

## File Structure

```
├── README.md (this file)
├── SETUP.md
├── esp32/
│   ├── plant_humidity_monitor-debug.yaml    # Real-time for testing
│   ├── plant_humidity_monitor-live.yaml     # 2s refresh for calibration
│   ├── plant_humidity_monitor-prod.yaml     # 15-min intervals (use this)
│   └── secrets.yaml.example                 # Copy and fill in your WiFi
└── home_assistant/
    ├── config/
    │   ├── configuration.yaml                # Add: homeassistant.packages
    │   ├── appdaemon/apps/apps.yaml          # (Optional)
    │   └── packages/plant_humidity_monitor/
    │       ├── core.yaml                     # Template sensors (calibration math)
    │       ├── user_config.yaml              # Controls and buttons
    │       └── automations.yaml              # Automations
    └── dashboard/
        ├── dashboard_card-overview.yaml      # All four plants at a glance
        └── dashboard_card-control_panel.yaml # Calibration controls
```

## Tips & Gotchas

1. **Resistive sensors and peat don't mix well.** If readings drop to zero without reason, it's air pockets. Not a bug — a hardware limitation. Upgrade to capacitive.
2. **Cable length matters.** Calibrate with your actual cable setup, not on the test bench.
3. **ADC2 pins are unusable with WiFi.** Stick to GPIO1-10 for sensor readings.
4. **Compile locally.** 3 minutes vs 10+ minutes changes how much you're willing to experiment.
5. **AI helps and sometimes hallucinates.** It made this project possible and also described the wrong sensor physics for two paragraphs. Verify things that matter.

## License

Personal project, shared freely. Do with it what you like.

## See Also

- [ESPHome Docs](https://esphome.io/)
- [Home Assistant Docs](https://www.home-assistant.io/docs/)
- [The Soil Sensor Saga](https://noawinged.me/entry/soil-sensor-saga) — the full story
