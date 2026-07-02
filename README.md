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
   - Subscribes to `+/data/+` MQTT topics to automatically log incoming room data into InfluxDB.
   - Periodically queries InfluxDB (every 2 seconds) for the last known states of `led`, `projector`, and `ac`, republishing them to control topics to ensure hardware state synchronization.
   - Daily schedule synchronizer to publish schedule rules.
2. **Dashboard**:
   - Built with FlowFuse Dashboard 2.0 widgets.
   - Exposes gauges, interactive graphs, and controllers for all room attributes.
   - Uses local flow context to reconstruct incoming scalar topics back into unified telemetry records for live UI widget updates.

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

---

## 5. Deployment Instructions

1. Copy the contents of `flows.json` from this folder.
2. In the Node-RED editor, go to **Menu** -> **Import**.
3. Paste the JSON data and select **Import to: New flow** or **Replace current flow** to clean up previous duplicates.
4. Verify the MQTT Broker config is set to `localhost:1883`.
5. Deploy the flow. The HMI will be accessible at: `http://localhost:1880/dashboard`
