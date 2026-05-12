# Sonoff Valve Smart Sprinkler — Setup Guide

Two Sonoff valves (`valve.backyard_tap` and `valve.grass`) with real-time
runtime and volume tracking. No Raspberry Pi, no REST API.

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Sonoff valves in HA | eWeLink cloud or SonoffLAN (local push) |
| HACS | For the two custom cards |
| `custom:button-card` | HACS → Frontend |
| `custom:bubble-card` | HACS → Frontend |
| Weather entity | Default: `weather.home` |
| Temperature sensor | Default: `sensor.outdoor_temperature` |
| Hero image | Place a JPEG at `/config/www/images/sprinkler.jpeg` |

---

## 2. Files overview

| File | Purpose |
|---|---|
| `dashboard.yaml` | Lovelace card — paste into a manual card |
| `scripts.yaml` | Run-both sequence + all-off using `valve.*` services |
| `template.yaml` | Active zone, last ran, next run, adjusted runtime, rain skip |
| `sensors.yaml` | Integration sensors: flow rate → cumulative litres |
| `utility_meter.yaml` | Daily + monthly water resets |
| `input_boolean.yaml` | Manual rain skip toggle |
| `input_datetime.yaml` | Last-run timestamp |
| `automations.yaml` | Schedule, safety shutoff, rain detection, daily reset |

---

## 3. Water volume tracking — how it works

```
sensor.backyard_tap_volume_flow_rate  (L/min, instantaneous)
        │
        ▼ integration (Riemann sum × time in minutes)
sensor.backyard_tap_volume_total      (L, cumulative, never resets)
        │
        ├─▶ sensor.backyard_tap_water_daily    (resets midnight)
        └─▶ sensor.backyard_tap_water_monthly  (resets 1st of month)
```

Same pipeline for `grass`.

### If your flow rate sensor is NOT in L/min

Open `sensors.yaml` and change `unit_time`:
- `L/h` → `unit_time: h`
- `m³/h` → `unit_time: h` (values will be in m³ — adjust display labels)

### If your sensors are already cumulative (total litres, not rate)

Skip `sensors.yaml` entirely and point `utility_meter.yaml` source directly
at `sensor.backyard_tap_volume_flow_rate` / `sensor.grass_volume_flow_rate`.

---

## 4. Add to `configuration.yaml`

```yaml
script:         !include scripts.yaml
sensor:         !include sensors.yaml
template:       !include template.yaml
input_boolean:  !include input_boolean.yaml
input_datetime: !include input_datetime.yaml
utility_meter:  !include utility_meter.yaml
```

Automations: append `automations.yaml` to your existing automations file,
or import each one manually via the HA UI.

---

## 5. Add the dashboard card

Settings → Dashboards → Edit → Add Card → Manual → paste `dashboard.yaml`.

---

## 6. Replace placeholder entities

Two template sensors reference generic entity IDs — update them in
`template.yaml` if needed:

| Placeholder | Replace with |
|---|---|
| `sensor.outdoor_temperature` | your outdoor temp sensor |
| `weather.home` | your weather entity (if different) |

---

## 7. Restart Home Assistant

Developer Tools → YAML → Check Configuration → Restart

---

## Card behaviour

| Action | Result |
|---|---|
| **Tap** valve card | Opens that valve (`valve.open_valve`) |
| **Hold** valve card | Closes that valve (`valve.close_valve`) |
| **Run Both Valves** | Runs Backyard Tap then Grass sequentially, weather-adjusted duration |
| **All Off** | Closes both valves immediately |
| **Emergency All Off** | Same — large red button for panic situations |

Each valve card shows three pills: **⏱ runtime today** · **💧 litres today** · **📅 litres this month**.

The "Run Both" script calls `sprinkler_all_off` before opening each valve
so two valves can never be open simultaneously.
