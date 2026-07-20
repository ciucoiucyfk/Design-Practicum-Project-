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
*   **Charge Controller:** MPPT IC (e.g., Consonance CN3791 for 6V panels).
*   **Battery Array:** 10,000mAh Lithium-Ion (18650 array).
*   **Voltage Regulation:** Ultra-low Quiescent Current (Iq) LDO (e.g., HT7333).
*   **Power Switching:** Logic-Level N-Channel MOSFETs (e.g., IRLML2502) to physically cut power to power-hungry sensors during deep sleep.

**Extended Micro-Climate Sensor Array:**
*   **BME680 (I2C):** Temperature, Humidity, Barometric Pressure, and VOCs.
*   **AS3935 (I2C/SPI):** Lightning sensor IC + MA5532 antenna coil (detects strikes up to 40km).
*   **NEO-6M GPS (UART):** GPS module (power-gated via MOSFET, used only for initial sync).
*   **Anemometer (GPIO Interrupt):** Wind speed pulse counter.
*   **Pluviometer (GPIO Interrupt):** Rain gauge tipping bucket.
*   **PIN Diode Array (ADC):** Beta/Gamma radiation detection via BPW34 + LM358 Op-Amp.
*   **INMP441 (I2S):** MEMS Digital Microphone for Acoustic AI (thunder, rain intensity, chainsaw detection).
*   **PMS5003 (UART):** Particulate Matter laser scatter sensor for PM2.5/PM10 (wildfire smoke, smog). Heavily power-gated via MOSFET.
*   **AS5600 (I2C):** Magnetic rotary encoder attached to a wind vane for Absolute Wind Direction.
*   **VEML7700 (I2C):** High-precision Ambient Light / UV sensor for calculating Solar Irradiance and cloud density.
*   **SHT20 (I2C via RS485):** Waterproof soil moisture and temperature probe.

## 2. Slave-to-Slave Mesh Instruction Set
Slave Nodes must autonomously manage the mesh without the Master. To do this, they exchange heavily formatted structs under the **SPIN (Sensor Protocols for Information via Negotiation)** routing rules.

### Allowed Slave-to-Slave Transmissions
1.  **HELLO Beacons:** Sent every few hours to maintain the Distance-Vector routing table and synchronize cryptographic nonces.
2.  **SPIN_ADV (Advertisement):** When a Slave's TinyML model detects an anomaly (e.g., a flash flood), it does *not* broadcast the heavy data immediately. It broadcasts a 3-byte ADV packet saying "I have new anomaly data."
3.  **SPIN_REQ (Request):** If a neighboring Slave has not seen this anomaly yet, it replies with a REQ packet.
4.  **SPIN_DATA (Data Payload):** The original Slave finally sends the encrypted anomaly payload.
5.  **MESH_WARNING:** A specific peer-to-peer warning instructing a neighboring node to wake up and increase its polling rate because a severe storm is approaching from its vector.

### Slave-to-Slave Packet Formatting
```cpp
// Peer-to-Peer Message Types
enum MeshCommand {
    P2P_HELLO         = 0x10, // DV Routing update
    P2P_SPIN_ADV      = 0x11, // I have new anomaly data
    P2P_SPIN_REQ      = 0x12, // Please send me that anomaly data
    P2P_SPIN_DATA     = 0x13, // Here is the encrypted anomaly data
    P2P_MESH_WARNING  = 0x14  // High alert: Storm approaching from my vector
};

// Standard P2P Header (Unencrypted for fast routing)
struct P2P_Header {
    uint8_t sender_mac[6];
    uint8_t msg_type; // Maps to MeshCommand enum
    uint8_t hop_metric;
};
```
*How they do it:* The SPIN protocol strictly prevents the "broadcast storm" problem. A node *only* transmits data if a neighbor explicitly requests it via `P2P_SPIN_REQ`.

## 3. The Master RPC Listener (Command Execution)
Unlike Peer-to-Peer instructions, the Slave Node runs a persistent **Remote Procedure Call (RPC) Listener Task** for Master Node commands.
*   When a LoRa packet arrives addressed to this Slave (or a global broadcast), the Slave decrypts the payload (AES-128-CTR). 
*   If valid, the Slave executes the instruction:
    *   *Data Polling:* The Master requests data (`CMD_SEND_TELEMETRY`); the Slave wakes, pulls the 1D Tensor, encrypts it, and replies.
    *   *Parameter Update:* The Master sends `CMD_SET_SLEEP_MS`; the Slave updates Non-Volatile Storage (NVS) and alters its RTOS timer instantly.

## 4. Master-to-Slave Instruction Set Structure
Every Slave Node possesses a predefined C-Struct dictionary of acceptable commands it will obey from the Master Node:
```cpp
enum RpcCommand {
    CMD_FORCE_INFERENCE   = 0x01, // Wake up and run XGBoost model now
    CMD_SEND_TELEMETRY    = 0x02, // Send raw telemetry to Master
    CMD_UPDATE_THRESH     = 0x03, // Change ML anomaly threshold (0.0 to 1.0)
    CMD_SET_SLEEP_MS      = 0x04, // Change deep sleep duration
    CMD_DISABLE_PERIPH    = 0x05, // Cut MOSFET power to a broken sensor (e.g., PM2.5 fan broken)
    CMD_REBOOT            = 0x06  // Trigger ESP.restart()
};
```

## 5. Autonomous Fallback (Survival Mode)
Because the network is a Hybrid Mesh, the Slave Nodes do not become useless if the Master Node goes offline.
*   **Connection Loss:** If a Slave Node fails to receive a Master Node poll for 24 hours, it enters Autonomous Mode.
*   **Fallback Behavior:** The Slave continues to route packets via P2P. It relies purely on the `MeshCommand` instructions to keep neighbors updated on severe weather, allowing the macro-region to react to storms even if the global Base Station is physically destroyed.
