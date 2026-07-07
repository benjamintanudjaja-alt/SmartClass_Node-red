# Node-RED Smart Classroom Integration Flow

This folder contains the complete, integrated Node-RED flow file combining the **Production Main Flow** (data logging, InfluxDB persistence, and control state syncer) with the **HMI Dashboard Flow** (interactive visualization using `@flowfuse/node-red-dashboard` 2.0).

---

## 1. Flow File
The merged, production-ready flow is located in [flows.json](file:///c:/Users/benja/Documents/PlatformIO/Projects/smartbuilding-Serial-Master-Resistive/node-red/flows.json).

---

## 2. Architecture & Tabs

The flow is organized into two primary tabs:
1. **Flow 1 (Main Flow)**:
   - The production backend engine.
   - Subscribes to `+/+/+` MQTT topics to automatically log incoming room data and commands into InfluxDB.
   - Periodically queries InfluxDB (every 2 seconds) for the last known states of `led`, `projector`, and `ac`, republishing them to control topics to ensure hardware state synchronization.
   - Daily schedule synchronizer to publish schedule rules.
2. **Dashboard**:
   - Built with FlowFuse Dashboard 2.0.
   - Uses a custom Single-Page Application (SPA) `ui-template` to dynamically render classrooms directly from InfluxDB query responses.
   - Exposes interactive modals for room control and historical trends.

---

## 3. MQTT Topic Structure

Topics follow the production generic schema:
* **Telemetry Data (Publish)**: `<room>/data/<field>`
* **Control Commands (Subscribe)**: `<room>/control/<device>`

### Mappings Table
| Telemetry Field | Control Device | Payload Type | Description | Dashboard Equivalent |
| :--- | :--- | :--- | :--- | :--- |
| `temp` | - | Float (e.g. `24.5`) | Average room temperature | `tempAvg`, `temp1-4` |
| `co2` | - | Int | Carbon Dioxide level (ppm) | `co` gauge |
| `lux` | - | Int | Light level (lux) | `luxAvg`, `lux1-4` |
| `human` | - | Int (`0` / `1`) | Human presence occupancy status | `isOccupied` indicator |
| `led` | `led` | Int (`0` / `1`) | General light relay state & command | Local lamps `lamp1-4` |
| `projector` | `projector` | Int (`0` / `1`) | Projector state & command | `Projector control` |
| `ac` | `ac` | String (8-char) | AC command & state | `AC control` |
| `alert` | - | String (8-bit) | Alarm bitmask (`temp,co2,lux,human,led,proj,ac,presence`) | HMI Status LED |
| `active` | `schedule` | Int (`0` / `1`) | Keepalive status heartbeat (LWT = `0`) | MQTT / Status indicator |

---

## 4. Device Control Details

### A. Lights / LED (`control/led`)
* **Payload**: `"1"` (ON) or `"0"` (OFF).
* **Behavior**: Dashboard individual lamp switches locally maintain their 4-channel states (`1010`). Toggling any channel ON publishes `"1"` (ON) to turn the system lights ON. Turning all channels OFF publishes `"0"` (OFF).

### B. Projector (`control/projector`)
* **Payload**: `"1"` (ON) or `"0"` (OFF).

### C. Air Conditioner (`control/ac`)
* **Payload**: 8-character string formatting `PPTTFFSS`
  - `PP`: Power (`00` for OFF, `01` for ON)
  - `TT`: Target Temperature in °C (e.g. `18` to `30`)
  - `FF`: Fan Speed (e.g. `00` default)
  - `SS`: Swing Mode (e.g. `00` default)
* **Example**: `"01180000"` turns the AC ON at 18°C. `"00180000"` turns the AC OFF.

## 5. Dashboard Architecture & Features

The HMI is built as a dynamic Single-Page Application (SPA) using a single `@flowfuse/node-red-dashboard` `ui-template` powered by Vue 3 and Vuetify.

### A. Core Features:
* **Dynamic Classroom Grid**: Automatically discovers active classrooms (e.g., `HD01`, `HD02`, `HD04`, `R2A`) by querying InfluxDB database records. New rooms are rendered as cards dynamically without code modification.
* **5-Minute Telemetry Timeout**: If a room does not report telemetry data for more than 5 minutes, its sensors show `-` values, status dots turn grey, and the entire card dims to indicate it is offline/inactive.
* **Device Status LEDs**: Displays state indicators for Lamps (`led_data`), AC (`ac_data`), and Projectors (`projector_data`) using color codes (Green = ON, Red = OFF, Grey = Inactive/Timeout).
* **Connection Status Bar**: Displays real-time LED connection indicators for MQTT, Cloud Connection, and **InfluxDB** (updates to Red if database queries error out).
* **Interactive Control & History Modal**:
  - Direct control toggles for LED lights and Projectors.
  - Sliders for AC target temperature (sending standard 8-character commands).
  - Interactive SVG historical trend charts (Temperature, Lux, and CO2) showing data from InfluxDB.
  - Graphs feature formatted X-axis time coordinates, Y-axis units (`°C`, `Lx`, `ppm`), and high-accuracy hover points displaying exact value and time tooltips.
  - **Timeframe Selector**: Toggle chart between presets (`Last Day`, `Last Week`, `Last Month`) or select a custom timeframe using native `datetime-local` start/end date-time inputs.

---

## 6. Deployment Instructions

1. Copy the contents of `flows.json` from this folder.
2. In the Node-RED editor, go to **Menu** -> **Import**.
3. Paste the JSON data and select **Import to: New flow** or **Replace current flow** to clean up previous duplicates.
4. Verify the MQTT Broker config is set to `localhost:1883`.
5. Deploy the flow. The HMI will be accessible at: `http://localhost:1880/dashboard`

---

## Changelog

### v3.0 (Current) - Dynamic InfluxDB & Interactive Trends Overhaul
* **Database Polling**: Migrated dashboard data source from direct MQTT subscriptions to periodic InfluxDB polling (every 5 seconds) to handle arbitrary rooms dynamically.
* **Removed Obsolete Controls**: Removed light calibration controls and individual device validations (shifted logic to ESP).
* **Interactive SVG Charts**: Implemented custom SVG charting inside the template containing X/Y axis labels, Y-axis units (`°C`, `Lx`, `ppm`), and hover-sensitive transparent overlay targets for instant point-specific tooltips.
* **Custom Start/End Picker**: Replaced duration slider with HTML5 `datetime-local` input elements for custom timeframe selection.
* **InfluxDB Connection LED**: Added top-bar LED connection status tracking node execution health.

### v2.0 - Measurement Separation Update
* **Ingestion Logic**: Updated Flow 1 to split incoming messages into `classroom_data` (for telemetry values) and `classroom_control` (for application control commands).
* **Target Schema**: Allowed integer flags (`led_data`, `ac_data`, `projector_data`) to write to InfluxDB.

### v1.0 - Initial Release
* **Static Dashboard**: Standard Node-RED Dashboard flow containing hardcoded grid widgets for `HD01` classrooms.
* **Validation & Calibration**: Integrated client-side lux calibration and device status validation flow.
