# Quick Setup Guide

This is the fast path if you already have ESP32 and Home Assistant experience.

## Hardware

- ESP32-S3-DevKit-C-1 + 4x soil moisture sensors + Dupont cables
- Connect sensors to GPIO1, GPIO2, GPIO4, GPIO5 (ADC)
- Connect power switches to GPIO38, GPIO39, GPIO40, GPIO41

## ESP32 Firmware

```bash
pip3 install esphome
cd esp32
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your WiFi
esphome run plant_humidity_monitor-prod.yaml
```

(Use `debug.yaml` for testing, `live.yaml` for calibration, `prod.yaml` for daily use)

## Home Assistant

1. **Enable packages** in `~/.homeassistant/config/configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   (Add this line if it doesn't exist. Restart HA after.)

2. Copy package: `cp -r home_assistant/config/packages/plant_humidity_monitor ~/.homeassistant/config/packages/`

3. Restart HA again

4. Check **Settings → Devices & Services** for ESPHome device + raw voltage sensors

5. Open control panel dashboard to calibrate:
   - Put each sensor in dry soil, record voltage
   - Water thoroughly, record voltage 15min later
   - Set these in the sliders

Done. Watch **Current Moisture** card for readings.

## Calibration Values (Default)

These are from testing with peat soil and ~2.5m cables:

| Plant | Dry (V) | Wet (V) | Alert Below |
|-------|---------|---------|-------------|
| 1 (Nephrolepis) | 2.11 | 0.88 | 50% |
| 2 (Adiantum) | 2.19 | 1.1.35 | 60% |
| 3 (Asplenium) | 2.01 | 1.15 | 40% |
| 4 (Phlebodium) | 1.98 | 1.07 | 40% |

Your values **will be different** — cable length, soil type, sensor variance all matter.

## Dashboard Cards

Copy to Lovelace:
```bash
cp home_assistant/dashboard/*.yaml ~/.homeassistant/config/lovelace/
```

Or import manually into the UI.

## Troubleshooting

**Moisture dropping impossibly fast (90% → 0% in hours)?** Peat soil shrinks and creates air pockets around the probes. Press sensors firmly back into fresh soil with solid contact.

**Readings way off?** Calibrate with actual cables in place.

See README.md for more details.
