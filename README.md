# Plant Humidity Monitor

A Home Assistant integration for real-time soil moisture monitoring across multiple plants using ESP32 and resistive soil sensors.

**Read the full story:** [The Soil Sensor Saga](https://noawinged.me/entry/soil-sensor-saga) — a blog post about building this system, transition, and discovering that sometimes the answer is far simpler than you think.

## Hardware Setup

### What You Need

- **ESP32-S3-DevKit-C-1** — the brain (~$10)
- **4x Resistive Soil Moisture Sensors** — the eyes in the dirt (~$5)
- **Dupont Jumper Cables** — connectors (~$2)
- **Home Assistant** — automation and history (something you likely have)

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

**Why power switches?** Sensors powered continuously cause electrolysis in soil, degrading it. This setup powers each sensor for only 2 seconds during measurement — about 48 seconds per day instead of 24 hours continuously.

## Installation

### 1. Flash the ESP32

```bash
pip3 install esphome
cd esp32/
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your WiFi credentials
esphome run plant_humidity_monitor-prod.yaml
```

Use the `plant_humidity_monitor-prod.yaml` (15-minute intervals) for daily use. The other two are for development:
- `debug.yaml` — real-time voltage for hardware testing
- `live.yaml` — 2-second refresh for calibration

### 2. Add Home Assistant Configuration

**First:** Make sure your `~/.homeassistant/config/configuration.yaml` has packages enabled:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

(If it's not there, add it. If the `homeassistant:` section already exists, just add the `packages:` line.)

**Then:** Copy the Home Assistant config package:

```bash
cp -r home_assistant/config/packages/plant_humidity_monitor \
  ~/.homeassistant/config/packages/
```

Restart Home Assistant. You should see four raw voltage sensors (`sensor.soil_moisture_1_raw`, etc.) appear in **Settings → Devices & Services**.

### 3. Calibration

The system sends **raw voltages** to Home Assistant. You define the math in software:

- **Air / Dry soil:** ~3.14V / ~2.10V
- **Water / Wet soil (just watered):** ~1.05 / ~1.15V
- **Linear mapping:** convert volts to percentage

For each plant:
1. Put sensor in dry soil from your pot. Record the voltage.
2. Water thoroughly. Record the voltage 15 minutes later.
3. Set these in the Home Assistant sliders (`Plant X Dry/Wet Limit`).

**⚠️ Calibrate with actual cables in place.** Cable length affects voltage — a 2.5m cable reads differently than 0.5m.

### 4. Dashboard

Copy the dashboard cards:

```bash
cp home_assistant/dashboard/*.yaml \
  ~/.homeassistant/config/lovelace/
```

Or configure manually in the HA UI. The key template sensors are `NS Plant 1/2/3/4` — they convert raw voltage to moisture percentage.

## Configuration Files

### ESP32 Configuration (`esp32/plant_humidity_monitor-prod.yaml`)

- **15-minute measurement cycle** — balances battery life and data freshness
- **Power management script** — safely powers sensors on/off
- **Raw voltage output** — Home Assistant handles calibration logic

Edit for your setup:
- Change `interval: 15min` if you want more/less frequent readings
- Add more sensors by duplicating the GPIO switch and ADC sensor blocks

### Home Assistant Core (`home_assistant/config/packages/plant_humidity_monitor/core.yaml`)

**Template sensors** that convert raw voltage to percentage:
- Each plant has its own calibration (dry/wet limits)
- Fertilizer offset slider for multi-day tracking
- Debug toggle to show raw voltage instead of percentage
- Icon changes to water-alert when moisture drops below thresholds

Thresholds per plant:
- Plant 1 & 4: alert below 50%
- Plant 2: alert below 60%
- Plant 3: alert below 40%

Edit these numbers based on your plant species' needs.

### Home Assistant User Config (`home_assistant/config/packages/plant_humidity_monitor/user_config.yaml`)

Sliders and buttons for:
- **Debug mode toggle** — show voltage instead of percent
- **Fertilizer offset** — account for nutrient conductivity changes
- **Calibration sliders** — dry/wet voltage limits per plant
- **Save/restore buttons** — snapshot and restore profiles

## Troubleshooting

### Sensors show moisture dropping impossibly fast

**Example:** You water thoroughly (90% moisture), but within 3 hours the reading drops to 30%, and by evening it's at 0%, even though the soil is still wet.

**Root cause:** Peat soil shrinks as it dries, pulling away from the sensor probes and creating microscopic air pockets. The sensor measures dielectric permittivity. When there's an air gap around the probe, it thinks the soil is dryer.

**Fix:** Press each sensor firmly into soil, making sure the probes have full, solid contact. This is not a bug — it's just physics.

### Sensor readings drift over time

Soil resistance changes as it dries. That's why we don't do "set it once" calibration. The controls in Home Assistant let you adjust on the fly. For long-term tracking, use the fertilizer offset control or re-calibrate every few weeks.

## Why This Design

### Raw voltage in firmware, calibration in HA

I wanted to avoid putting all the math in the ESP32. That means flashing new firmware every time you re-calibrate. This design sends raw voltage and does the math in HA instead.

**Why:** Calibration changes. Soil type changes. You add a plant or swap sensors. Doing the logic in HA means you tweak a slider instead of reflashing.

### 15-minute intervals with power management

Battery projects use deep sleep. This one is plugged in, so we don't. But we still power-gate the sensors to avoid electrolysis. 15 minutes is long enough that you see trends, short enough that you catch problems before they're serious.

### Four plants, hardcoded

This started as a one-off for four ferns. Generalizing to N plants would add complexity. If you need more, duplicate the sensor blocks in the YAML and the template sensor blocks in core.yaml.

## File Structure

```
├── README.md (this file)
├── esp32/
│   ├── plant_humidity_monitor-debug.yaml    # Real-time for testing
│   ├── plant_humidity_monitor-live.yaml     # 2s refresh for calibration
│   ├── plant_humidity_monitor-prod.yaml     # 15-min intervals (use this)
│   └── secrets.yaml.example                 # Copy and fill in your WiFi
└── home_assistant/
    ├── config/
    │   ├── configuration.yaml                # Add: homeassistant.packages.plant_...
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

1. **Peat soil shrinks.** When readings drop to zero without reason, it's probably air pockets. Not a bug.
2. **Cable length matters.** Voltage changes with resistance. Calibrate with your actual cable setup.
3. **Two types of meters measure different things.** A capacitive sensor and a resistive meter won't agree. Both are right for their measurement type.
4. **Iteration speed matters.** Local compilation on your machine is 3× faster than on Home Assistant Green. Set that up first.
5. **AI made this possible.** Without being able to ask questions and get real answers, this project would've taken weeks. Now it's a weekend.

## License

This is a personal project shared freely. Do with it what you like.

## See Also

- [ESPHome Docs](https://esphome.io/)
- [Home Assistant Docs](https://www.home-assistant.io/docs/)
- [The Soil Sensor Saga](https://noawinged.me/entry/soil-sensor-saga) — the full story of building this
