# Web Interface & Global Configuration (Master Node)

The Master Node acts as the sole bridge between the localized LoRa mesh and the external world (Base Station/Internet). It hosts an interactive Web Dashboard accessible via WiFi/Ethernet.

## 1. Dashboard Layout & Content
The web interface is built using a lightweight framework (e.g., React/Vite served via an external server, or a lightweight HTML/JS payload served directly from the Master Node's SPIFFS if it acts as an AP).

### A. The World Map Interface (Geospatial View)
*   **Mapping Engine:** Utilizes Leaflet.js or Mapbox GL.
*   **Dynamic Node Markers:** Each Slave Node is represented by a marker plotted using its locally acquired GPS coordinates (sent during its initial network integration).
*   **Heatmaps:** Overlays interpolated heatmaps for Temperature, Humidity, and VOCs based on the Master Node's spatial inferencing engine.
*   **Event Alerts:** If a slave node broadcasts an "Exception-Based Alert" (e.g., Severe Storm), its marker flashes red, and a vector arrow indicates wind direction and storm trajectory.

### B. Node Management Console
Clicking a node marker opens a side-panel displaying:
*   **Current Telemetry:** Last known Temp, Battery %, Wind RPM.
*   **Hardware Status:** Solar charging rate, ULP sleep state.
*   **Link Quality:** RSSI and SNR of the last hop.

## 2. Modifying Network Parameters (The RPC Engine)
The web interface acts as the frontend for the Master Node's Remote Procedure Call (RPC) engine. Through the UI, an admin can push configurations to the entire mesh or single nodes.
*   **Global Configuration Broadcast:** 
    *   *Example:* Update the TinyML anomaly threshold network-wide. The Master broadcasts `[CMD: GLOBAL_SET_THRESH, VAL: 0.85]`. 
*   **Targeted Parameter Tuning:**
    *   *Force Wake:* Instruct a specific slave to abort Deep Sleep and maintain an active radio link for real-time debugging.
    *   *Sensor Disable:* If a node's BME680 breaks and starts spamming the network with false anomalies, the admin can send `[CMD: DISABLE_BME680, TARGET: MAC_ADDR]`.

## 3. Efficient Data Collation Strategy
To update the world map without causing a LoRa packet collision storm, the Master Node uses a strictly managed **Staggered Polling Queue**.
*   **The Queue:** The Master maintains a list of all active slaves.
*   **The Polling Loop:** Every 15 minutes, the Master sends a targeted `DATA_REQ` to Node 1. It waits for the response (with a 2-second timeout). It then sends a `DATA_REQ` to Node 2.
*   **Interpolation on Missing Data:** If Node 3 fails to respond, the Master does not stall. It skips Node 3, and the Map Interface uses Spatial Inferencing to estimate Node 3's environment based on Nodes 2 and 4.
