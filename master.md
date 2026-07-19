# Master Node Architecture

The Master Node is the apex of the network topology. Unlike Slave Nodes, which prioritize micro-amp power efficiency, the Master Node is connected to a stable Base Station power supply and acts as the network's brain and gateway.

## 1. Hardware & Component List (BOM)
Because the Master Node hosts an interactive web server (React/Leaflet), collates data from the entire mesh, and runs spatial inferencing algorithms, it requires significantly more compute and memory overhead than the micro-power Slave Nodes. 

### Comprehensive Master Node Component List (BOM)
**Compute & Core Processing (The Dual-ESP or Single PSRAM Setup):**
*   **Application Processor:** ESP32-S3 WROOM (with 8MB PSRAM and 16MB Flash). This runs the Asynchronous Web Server, Spatial Interpolation, and SD Card datalogging. 
*   *(Optional Decoupling):* The user can opt for a "Dual-ESP" master board where ESP #1 handles the real-time LoRa Mesh MAC, and ESP #2 handles the WiFi Webserver, communicating via UART. However, a single PSRAM-equipped ESP32-S3 utilizing FreeRTOS dual-core pinning is fully capable of doing both.

**Communications & Connectivity:**
*   **LoRa Transceiver:** SX1262 SPI Module with a high-gain 868MHz 8dBi/10dBi Antenna.
*   **Base Station Backhaul:** Native ESP32 WiFi. The Master Node acts either as a WiFi Access Point (AP) for direct phone/laptop connection in the field, or connects to a local WiFi router. **No cellular/LTE is permitted.**

**Power Subsystem (Identical to Slave Node):**
Because the Master Node is subjected to the same strict power constraints as the slaves, it cannot rely on heavy Linux SBCs.
*   **Solar Panel:** 10W Monocrystalline.
*   **Charge Controller:** MPPT IC (e.g., CN3791).
*   **Battery Array:** 10,000mAh Lithium-Ion (18650 array).
*   *Note:* The Master Node will draw slightly more average current than a slave because the WiFi radio must wake up to serve web requests, but FreeRTOS Modem Sleep will keep it well within the 10W solar budget.

**Storage & Datalogging:**
*   **Primary Storage:** MicroSD Card Module (wired via high-speed SDIO, not standard SPI). This SD card hosts the static Web Assets (HTML, React/Vite bundles, Leaflet.js files) and logs the network's historical telemetry in CSV or SQLite format.

**Misc / Timing:**
*   **RTC Module:** DS3231 Precision Real-Time Clock (I2C) with a CR2032 coin cell battery to timestamp mesh data precisely without relying on an internet NTP server.

## 2. The RPC Command Center
The Master Node maintains a master list of all instruction sets available on Slave Nodes. 
*   **Command Injection:** The Master can wrap any command into an AES-128-CTR encrypted packet and inject it into the mesh.
*   **Instruction Set Examples:**
    *   `0x01`: Force localized TinyML Inference.
    *   `0x02`: Report precise battery voltage.
    *   `0x03`: Update sleep interval.
    *   `0x04`: Reboot node.

## 3. Spatial Inferencing & Regional Modeling
While Slave Nodes perform *Temporal Inferencing* (analyzing their own local data over time to detect anomalies), the Master Node performs **Spatial Inferencing**.
*   **The Data Lake:** The Master polls and collates data from all slaves into an internal data structure.
*   **Spatial Interpolation (Kriging / IDW):** The Master calculates environmental variables for physical spaces *between* nodes. 
    *   *Example:* Node A reports 25°C. Node B (2km away) reports 21°C. The Master uses Inverse Distance Weighting (IDW) or a localized neural network to infer that the region exactly between them is likely 23°C, plotting a smooth gradient on the Web Map.
*   **Macro-Pattern Detection:** The Master correlates warnings. If Slave A, B, and C all trigger a "Wind Anomaly" consecutively over 10 minutes, the Master Node identifies this as a moving weather front (squall line) and calculates its velocity and trajectory, updating the Web Interface with an ETA for the base station.

## 4. Base Station Integration
*   The Master Node runs an HTTP REST API or an MQTT broker/client over WiFi. 
*   When the Web Interface requests data, it queries the Master Node's local cache via HTTP/MQTT, NOT the LoRa network directly, ensuring the UI is instantaneous and the LoRa network is completely shielded from web traffic.
