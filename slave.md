# Slave Node Architecture

With the introduction of the Master Node overlay, the standard sensor nodes in the field are now officially classified as **Slave Nodes**. While they maintain autonomous Peer-to-Peer routing capabilities, their command-and-control structure is subordinate to the Master Node.

## 1. Hardware & Objective
*   **Objective:** The Slave Node's sole purpose is to survive indefinitely on solar power, collect high-fidelity environmental data, run local temporal inferencing (TinyML), and route packets.
*   **Inter-Slave Communication:** Slaves are fully capable of talking to one another. If Slave A detects a microburst, it broadcasts that alert directly to Slave B and Slave C via the SPIN protocol. They do **not** need the Master's permission to share data or pass warnings among themselves.

### Comprehensive Slave Node Component List (BOM)
**Compute & Communications:**
*   **Microcontroller:** ESP32-S3 WROOM Module (Dual-core, vector AI instructions, ULP coprocessor, minimal PSRAM required).
*   **LoRa Transceiver:** SX1262 SPI Module (e.g., EByte E22-900M22S) tuned for 865–867 MHz.
*   **Antenna:** 868MHz 3dBi/5dBi Omnidirectional Dipole Antenna (SMA connector).

**Power Subsystem (100% Autonomous):**
*   **Solar Panel:** 10W Monocrystalline (12V or 6V depending on MPPT).
*   **Charge Controller:** MPPT IC (e.g., Consonance CN3791 for 6V panels, or TI BQ24650).
*   **Battery Array:** 10,000mAh Lithium-Ion (18650 array) or LiFePO4 pack.
*   **Voltage Regulation:** Ultra-low Quiescent Current (Iq) LDO (e.g., HT7333) or high-efficiency Buck-Boost (e.g., TPS63020) for 3.3V stable output.
*   **Power Switching:** Logic-Level N-Channel MOSFETs (e.g., IRLML2502) to physically cut power to GPS/Sensors during deep sleep.
*   **Battery Monitoring:** 100kΩ/100kΩ resistor divider bridged to an ADC pin.

**Environmental Sensor Array:**
*   **BME680 (I2C):** Micro-heated MEMS for Temperature, Humidity, Barometric Pressure, and VOC (Air Quality).
*   **AS3935 (I2C/SPI):** Lightning sensor IC + customized MA5532 antenna coil (detects strikes up to 40km).
*   **NEO-6M GPS (UART):** Standard GPS module with active ceramic antenna (used only rarely for location fixes).
*   **Anemometer (Wind Speed):** 3D-printed wind cups + High-resolution Optical/Magnetic Rotary Encoder (GPIO Interrupt/PCNT).
*   **Pluviometer (Rain Gauge):** DIY Tipping Bucket + Reed Switch + Hardware RC Debounce circuit (10kΩ resistor, 100nF ceramic capacitor).
*   **Radiation Detector:** Solid-state PIN Diode array (e.g., BPW34) + Peak detector/Amplifier circuit (e.g., LM358 Op-Amp) for Beta/Gamma detection.

**Enclosure & Passives:**
*   IP67/IP68 Weatherproof ABS/Polycarbonate Enclosure.
*   Gore-Tex or PTFE vents (to allow ambient air pressure/humidity to reach the BME680 while blocking liquid water).
*   I2C Pull-up Resistors (4.7kΩ).

## 2. The RPC Listener (Command Execution)
Unlike the Master Node which injects commands, the Slave Node runs a persistent **Remote Procedure Call (RPC) Listener Task** on Core 0.
*   When a LoRa packet arrives addressed to this specific Slave Node (or a broadcast to all nodes), Layer 5 hands the payload to the RPC Listener.
*   The Listener decrypts the payload (AES-128-CTR) and verifies the HMAC. 
*   If valid, the Slave executes the instruction set command. 
    *   *Data Polling Request:* If the Master requests current sensor data, the Slave immediately wakes Core 1, pulls the latest 1D Tensor from Layer 2, encrypts it, and transmits it back to the Master.
    *   *Parameter Update:* If the Master sends a new Sleep Interval (e.g., `[CMD: SET_SLEEP, VAL: 300s]`), the Slave updates its Non-Volatile Storage (NVS) and alters its RTOS timer instantly.

## 3. Autonomous Fallback (Survival Mode)
Because the network is a Hybrid Mesh, the Slave Nodes do not become useless if the Master Node goes offline.
*   **Connection Loss:** If a Slave Node fails to receive a `HELLO` beacon or a polling request from the Master Node for 24 hours, it enters Autonomous Mode.
*   **Fallback Behavior:** The Slave continues to route packets for its peers. It relies purely on the Exception-Based Broadcasting (SPIN protocol) outlined in `communications.md`. It will locally trigger alerts if its TinyML model detects a severe weather anomaly, allowing adjacent Slave Nodes to act on that localized warning even if the global Base Station is disconnected.

## 4. Slave Node Instruction Set Structure
Every Slave Node possesses a predefined C-Struct dictionary of acceptable commands it will obey from the Master Node:
```cpp
enum RpcCommand {
    CMD_FORCE_INFERENCE   = 0x01, // Wake up and run XGBoost model now
    CMD_SEND_TELEMETRY    = 0x02, // Send raw temp/hum/press/wind
    CMD_UPDATE_THRESH     = 0x03, // Change ML anomaly threshold (0.0 to 1.0)
    CMD_SET_SLEEP_MS      = 0x04, // Change deep sleep duration
    CMD_DISABLE_PERIPH    = 0x05, // Cut MOSFET power to a broken sensor
    CMD_REBOOT            = 0x06  // Trigger ESP.restart()
};
```
