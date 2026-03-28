# Node-RED Mock Pipeline — Technical Summary

This document covers the software stack, installation, configuration, and architecture of the Caelus Node mock data pipeline. This is the first committed stage of the project -- a fully working data pipeline running on mock sensor data, before any physical hardware is connected.

---

## What you need

### Software
- **Node.js** -- required by Node-RED, install from nodejs.org
- **Node-RED** -- install globally via npm:
  ```
  npm install -g node-red
  ```
- **FlowFuse Dashboard 2.0** -- install from inside Node-RED:
  - Menu → Manage Palette → Install
  - Search for `@flowfuse/node-red-dashboard`
  - Version used: 1.30.2
  - Note: the older `node-red-dashboard` package is deprecated and uses incompatible node types -- do not install that one, and if you have it installed remove it first
- **Mosquitto MQTT broker** -- install from mosquitto.org
  - On Windows, run as a service: `net start mosquitto`
  - Runs on `localhost:1883` by default, no configuration needed for local use

### Hardware (for this stage)
None. Everything runs on one laptop.

---

## How to run it

1. Start Mosquitto: `net start mosquitto`
2. Start Node-RED: `node-red` in a terminal
3. Open the Node-RED editor: `http://localhost:1880`
4. Import the flow JSON via Menu → Import → upload file
5. Deploy
6. Open the dashboard: `http://localhost:1880/dashboard/weather`
7. The sidebar navigation gives access to all pages including the receiver alert pages

---

## Flow architecture

The pipeline is split across two tabs in Node-RED: **Caleus Weather Station** and **Caleus Reciever**.

---

### Tab 1: Caleus Weather Station

This tab contains the full sensor pipeline, alert logic, and dashboard output.

```
[10x Inject nodes]
        |
   [Join node]
        |
[Timestamp & Station ID]
        |
   -----+----------+----------+
   |               |          |
[Pipeline_debug] [Dashboard  [Alert
                  Splitter]   Router]
```

**Inject nodes (10 total)**

One per sensor channel. Each fires automatically every 60 seconds and once on deploy. Payload type is `num`, topic is the field name string:

| Node name | Topic | Mock value |
|-----------|-------|------------|
| temp_celsius | temp_celsius | 24.9 |
| wind_ms | wind_ms | 6.9 |
| wind_direction_degree | wind_direction_degree | 225 |
| uv_index | uv_index | 9 |
| rainfall_mm | rainfall_mm | 2.3 |
| noise_level_db | noise_level_db | 59.6 |
| battery_percent | battery_percent | 67 |
| vibration_ms2 | vibration_ms2 | 2.3 |
| humidity_percent | humidity_percent | 65 |
| air_pressure_hpa | air_pressure_hpa | 1013.2 |

**Join node** (`Join_sensory_inputs`)

Waits for all 10 messages (count: 10, timeout: 10 seconds), then merges them into a single object keyed by `msg.topic`. One clean object per cycle.

**Timestamp & Station ID function**

Adds three fields to the merged object:
```javascript
msg.payload.timestamp = new Date().toISOString();
msg.payload.station_id = "CALEUS_001";
msg.payload.station_location = "placeholder_lat,placeholder_lon";
return msg;
```
Change `station_id` and `station_location` per physical device when real hardware is deployed.

**Output branches three ways:**

---

#### Branch 1: Pipeline_debug
Debug node, logs the full merged object to the Node-RED sidebar. Active by default, can be disabled without affecting the flow.

---

#### Branch 2: Dashboard Splitter

Function node with 9 outputs, extracts individual values and routes them to dashboard widgets:

| Output | Destination | Value sent |
|--------|-------------|------------|
| 1 | Temperature gauge | `d.temp_celsius` |
| 2 | Wind gauge | `d.wind_ms` |
| 3 | UV Index gauge | `d.uv_index` |
| 4 | Noise Level gauge | `d.noise_level_db` |
| 5 | Humidity gauge | `d.humidity_percent` |
| 6 | Air Pressure gauge | `d.air_pressure_hpa` |
| 7 | Wind Direction text | degrees + cardinal label e.g. `225° SW` |
| 8 | Temperature History chart | `{x: ISO timestamp, y: temp_celsius}` |
| 9 | Noise Level History chart | `{x: ISO timestamp, y: noise_level_db}` |

---

#### Branch 3: Alert Router

Function node with 4 outputs, packages each alert-relevant value with the full data object for context:

```javascript
var d = msg.payload;
function mk(v){return {payload:{value:v, data:d}};}
return [mk(d.uv_index), mk(d.wind_ms),
        mk(d.noise_level_db), mk(d.vibration_ms2)];
```

| Output | Goes to |
|--------|---------|
| 1 | UV Alert Levels switch |
| 2 | Wind Alert Levels switch |
| 3 | Noise Alert Levels switch |
| 4 | Vibration Alert Levels switch |

**Alert switches (4 total)**

Each evaluates `msg.payload.value` against tiered thresholds using `checkall: false` (first match only):

| Switch | Rules | Output routing |
|--------|-------|----------------|
| UV Alert Levels | ≥11, ≥8, ≥6, ≥3 | Output 1 (≥11) → Severe; Outputs 2-4 → UV alert builder |
| Wind Alert Levels | ≥24, ≥17, ≥10 | Output 1 (≥24) → Severe; Outputs 2-3 → Wind alert builder |
| Noise Alert Levels | ≥85, ≥65 | Both outputs → Noise/Police alert builder |
| Vibration Alert Levels | ≥8, ≥5, ≥2 | Outputs 1-2 → Severe; Output 3 → Vibration alert builder |

**Alert builder functions (5 total)**

Each builds a structured JSON packet and sets `msg.topic`:

- **Build UV Alert Packet** → topic `alerts/park/CALEUS_001/uv`
- **Build Wind Alert Packet** → topic `alerts/park/CALEUS_001/wind`
- **Build Vibration Alert Packet** → topic `alerts/park/CALEUS_001/vibration`
- **Build Noise/Police Alert Packet** → topic `alerts/police/noise/CALEUS_001`
- **Build Severe / Global Alert Packet** → topic `alerts/global/severe/CALEUS_001`

Each packet contains: `alert_type`, `level`, `value` (or `value_ms`, `value_db`, `value_ms2`), `station_id`, `location`, `timestamp`, `message`.

**From each alert builder, output goes to multiple destinations simultaneously:**

UV, Wind, Vibration and Severe builders wire to 4 nodes:
1. `🌳 MQTT Park Alert` -- publishes to `caleus/alerts/park`
2. `🏫 MQTT School Alert` -- publishes to `caleus/alerts/school`
3. `Alert_debug` -- logs to sidebar
4. `json-to-message` -- station alert log function

Noise/Police builder wires to 3 nodes:
1. `🚔 MQTT Police Report` -- publishes to `caleus/alerts/police`
2. `Alert_debug`
3. `json-to-message`

Severe builder additionally wires to:
1. `🚨 MQTT Global Severe` -- publishes to `caleus/alerts/severe`
2. Plus the same park, school, debug, and log nodes above

**MQTT Out nodes (4 total)**

All connected to broker config node `MQTT Broker - Station` (client ID: `caleus_station`, host: `localhost:1883`):

| Node | Topic | QoS | Retain |
|------|-------|-----|--------|
| 🌳 MQTT Park Alert | caleus/alerts/park | 1 | false |
| 🏫 MQTT School Alert | caleus/alerts/school | 1 | false |
| 🚔 MQTT Police Report | caleus/alerts/police | 2 | false |
| 🚨 MQTT Global Severe | caleus/alerts/severe | 2 | true |

**json-to-message function (station tab)**

Maintains a shared scrolling log of the last 10 alerts for the Active Alerts widget on the main dashboard page. Uses `flow` context variable `alert_log`:

```javascript
var log = flow.get('alert_log') || [];
var p = msg.payload;
var entry = new Date().toLocaleTimeString() + ' — ' +
            p.alert_type.toUpperCase() + ' | ' + p.level + ' | ' + p.message;
log.unshift(entry);
if (log.length > 10) log.pop();
flow.set('alert_log', log);
msg.payload = log.join('<br>');
return msg;
```

Wires to the `Active Alerts` ui-text node on the Weather Station page.

---

### Tab 2: Caleus Reciever

Simulates the receiving end -- what a device at a school, park, or police station would do with incoming MQTT alerts.

```
[MQTT In - park]    → [json-to-message (park)]    → [ui-text - Park Alert page]
[MQTT In - school]  → [json-to-message (school)]  → [ui-text - School Alert page]
[MQTT In - police]  → [json-to-message (police)]  → [ui-text - Police Alert page]
[MQTT In - severe]  → [json-to-message (severe)]  → [ui-text - Severe/Global page]
```

**MQTT In nodes (4 total)**

Subscribed topics and settings:

| Topic | QoS | datatype |
|-------|-----|---------|
| caleus/alerts/park | 2 | auto-detect |
| caleus/alerts/school | 2 | auto-detect |
| caleus/alerts/police | 2 | auto-detect |
| caleus/alerts/severe | 2 | auto-detect |

`datatype: auto-detect` means Node-RED automatically parses the incoming JSON string into a JavaScript object, so the downstream function node receives a proper object rather than a raw string -- no separate parsing step needed.

All four currently connect to broker config node `MQTT Broker - Receiver` (same localhost:1883, client ID left blank so Node-RED auto-generates a unique one). Using a different client ID from the station broker is important -- if both sides share a client ID, Mosquitto disconnects and reconnects them repeatedly, causing duplicate message delivery.

**json-to-message functions (4 total, one per channel)**

Each maintains its own separate alert log using a dedicated flow context variable. This prevents cross-channel contamination where an alert on one channel would appear in another channel's log:

```javascript
// park channel example
var log = flow.get('park_alert_log') || [];
var p = msg.payload;
var entry = new Date().toLocaleTimeString() + ' — ' +
    p.alert_type.toUpperCase() + ' | ' + p.level + ' | ' + p.message;
log.unshift(entry);
if (log.length > 10) log.pop();
flow.set('park_alert_log', log);
msg.payload = log.join('<br>');
return msg;
```

| Function | Context variable |
|----------|-----------------|
| Park channel | `park_alert_log` |
| School channel | `school_alert_log` |
| Police channel | `police_alert_log` |
| Severe channel | `severe_alert_log` |

**ui-text nodes (4 total)**

One per channel, placed in their respective dashboard page groups. Format field uses triple curly braces to render HTML line breaks: `{{{msg.payload}}}`.

---

## Dashboard structure

Dashboard runs at `http://localhost:1880/dashboard/weather`.

**ui-base:** `Caleus Weather Station UI`, path `/dashboard`

**Pages:**

| Page | Path | Content |
|------|------|---------|
| Weather Station | /weather | Main monitoring page |
| Park Alert | -- | Park channel alert log |
| School Alert | -- | School channel alert log |
| Police Alert | -- | Police channel alert log |
| Severe / Global Alert | -- | Severe channel alert log |

**Weather Station page groups:**
- *Current Conditions* -- six gauges plus wind direction text widget
- *Active Alerts* -- shared scrolling log of last 10 alerts across all types, newest first
- *History* -- temperature and noise level line charts, 60 points / 10 hour retention

**Gauge configuration notes for Dashboard 2.0:**

Gauges use specific properties that differ from the deprecated dashboard package. Key fields that must be correct:
- `gtype`: `gauge-34` (3/4 arc)
- `gstyle`: `rounded`
- `value` / `valueType`: `payload` / `msg`
- `segments`: array of objects, each with `from` (string), `color`, `text`, `textType: "label"`
- `min` / `max`: numbers not strings
- `sizeThickness`, `sizeGap`, `sizeKeyThickness`: arc visual weight

**Theme:**

A `ui-theme` node (`Theme Name`) is attached to the Weather Station page. The page `theme` field must reference a valid theme node ID -- leaving it empty or as `undefined` produces repeated console warnings on deploy.

---

## Known issues and planned rework

- The `json-to-message` function name is used for both the station alert log and all four receiver channel logs -- if debugging, check which tab the node belongs to
- Flow context resets on Node-RED restart, so alert logs are lost between sessions. A persistent context store will be needed for production
- Alert builder topics include station ID (`alerts/park/CALEUS_001/uv`) but MQTT Out nodes publish to flat topics (`caleus/alerts/park`). These need to be unified when multi-node routing is planned
- All four MQTT In nodes on the receiver tab use the same broker config as the station MQTT Out nodes. A separate broker config node (`MQTT Broker - Receiver`) exists in the config panel but is not currently wired. This should be corrected to avoid potential client ID conflicts when both tabs are active simultaneously
- The `uv_index` mock value is set to 9 for testing alert thresholds. Change back to 5.3 for realistic idle conditions