# Quick Setup Guide

Fast path if you already have ESP32 and Home Assistant experience.

> **Note:** This project is in the middle of a sensor upgrade — moving from resistive to capacitive sensors (Cytron Maker Soil Moisture). The configs work with both, but calibration values will differ. See README for context on why the switch is happening.

## Hardware

- ESP32-S3-DevKit-C-1 + 4x soil moisture sensors + Dupont cables
- Connect sensors to GPIO1, GPIO2, GPIO4, GPIO5 (ADC1 — stable with WiFi)
- Connect power switches to GPIO38, GPIO39, GPIO40, GPIO41

**Sensor type matters:** Resistive sensors (cheap fork-style) require direct soil contact and will drop to 0% if peat shrinks away from the probes. Capacitive sensors (Cytron Maker Soil Moisture recommended) are more forgiving. Both use the same pinout.

## ESP32 Firmware

```bash
pip3 install esphome
cd esp32
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your WiFi
esphome run plant_humidity_monitor-prod.yaml
```

Use `debug.yaml` for hardware testing, `live.yaml` for calibration, `prod.yaml` for daily use.

**Compile locally** (laptop, not HA Green) — ~3 minutes vs 10+ minutes per build.

## Home Assistant

1. **Enable packages** in `~/.homeassistant/config/configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. Copy package:
   ```bash
   cp -r home_assistant/config/packages/plant_humidity_monitor ~/.homeassistant/config/packages/
   ```

3. Restart HA. Check **Settings → Devices & Services** for raw voltage sensors.

4. Open control panel dashboard to calibrate:
   - Put each sensor in dry soil, record voltage
   - Water thoroughly, record voltage 15 min later
   - Set these in the sliders
   - **Always calibrate with actual cables in place** — cable length shifts voltage readings

Done. Watch the **Current Moisture** card for readings.

## Calibration Values (From v1 — Resistive Sensors, Peat Soil, ~2.5m Cables)

These are provided as reference only. Your values will differ based on sensor type, cable length, and soil mix.

| Plant | Dry (V) | Wet (V) | Alert Below |
|-------|---------|---------|-------------|
| 1 (Nephrolepis) | 2.11 | 0.88 | 50% |
| 2 (Adiantum) | 2.19 | 1.12 | 60% |
| 3 (Asplenium) | 2.01 | 1.15 | 40% |
| 4 (Phlebodium) | 1.98 | 1.07 | 40% |

Capacitive sensor values will be added after the hardware upgrade is complete.

## Dashboard Cards

```bash
cp home_assistant/dashboard/*.yaml ~/.homeassistant/config/lovelace/
```

Or import manually into the UI.

## Troubleshooting

**Moisture crashing to 0% within hours of watering?**
Resistive sensor air gap problem — peat shrinks as it dries, breaking contact with the metal probes. Press sensors firmly back into soil. Long-term fix: upgrade to capacitive sensors.

**Readings way off after moving cables?**
Recalibrate with actual cable lengths in place.

See README.md for full details.
