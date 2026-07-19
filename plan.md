# Micro-Climate & Environmental Mesh Node Architecture
## Overview
This document outlines the architecture, layers, and component costs for a decentralized, AI-driven weather and environmental monitoring mesh network. Each node operates on the edge, running quantized machine learning models and communicating via a LoRa-based Distance-Vector routing protocol.

## System Layers

### Layer 0: Power Distribution System
Handles energy harvesting, storage, and voltage regulation to support the ESP32 logic, radio, and sensors.

| Component | Function / Spec | Estimated Cost (INR) |
| :--- | :--- | :--- |
| **Solar Panel** | 10W, 12V or 5V Polycrystalline Panel for rapid recharging. | ₹550 |
| **Battery Pack** | 10,000mAh 3.7V Li-ion Pack (3-4 parallel 18650 cells). | ₹550 |
| **Charge Controller** | TP5100 or MPPT solar charge controller. | ₹240 |
| **Voltage Regulation** | High-efficiency buck converter + AP2112K 3.3V LDO. | ₹160 |
| **Battery Voltage Sensor** | Resistor voltage divider (ADC input). | ₹40 |
| **Total Layer 0 Budget** | | **₹1,540** |

### Layer 1: Physical Sensing & Hardware Abstraction (HAL)
Captures the physical environment using digital (I2C/UART) and mechanical (GPIO interrupt) interfaces.

| Component | Function / Interface Protocol | Estimated Cost (INR) |
| :--- | :--- | :--- |
| **Microcontroller** | ESP32-S3 (NodeMCU). Handles sensors, ML, and crypto. | ₹400 |
| **Temp, Hum, Pressure, AQI** | **BME680 Breakout** (I2C). Low-power MEMS sensor for 4 metrics. | ₹750 |
| **Windspeed** | **Rotary Encoder + 3D Printed Cups** (GPIO). Calculates RPM. | ₹120 |
| **Precipitation** | DIY Tipping Bucket + Magnetic Reed Switch (GPIO). | ₹160 |
| **Radiation (Low Power)** | **PIN Diode Radiation Sensor** (e.g., Pocket Geiger Type 5 or custom BPW34 array). Uses solid-state PIN diodes instead of high-voltage GM tubes to detect Beta/Gamma radiation at microwatt power levels. | ₹2,400 |
| **Localization** | NEO-6M GPS Module (UART). | ₹270 |
| **Lightning Detection** | AS3935 Digital Sensor (I2C/SPI). | ₹1,200 |
| **Total Layer 1 Budget** | | **₹5,300** |

### Layer 2: Data Preprocessing & Feature Engineering
* **Software Layer:** Executes on the ESP32.
* **Function:** Normalizes physical data arrays (e.g., scaling RPM, CPM, and AQI indices) into a 1D tensor specifically formatted for the TinyML model.
* **Cost:** ₹0 (Open Source)

### Layer 3: Edge Intelligence (TinyML Engine)
* **Software Layer:** Executes on the ESP32.
* **Function:** Runs a quantized ML model (e.g., XGBoost/CatBoost) using TensorFlow Lite for Microcontrollers to output micro-climate predictions (e.g., frost or storm probability) directly on the node.
* **Cost:** ₹0 (Open Source)

### Layer 4: Cryptographic Security
* **Software Layer:** Leverages ESP32 Hardware Acceleration.
* **Function:** Encrypts the TinyML output and sensor payload using AES-128-CTR and generates an HMAC-SHA256 signature to prevent spoofing and data injection.
* **Cost:** ₹0 (Included in MCU hardware)

### Layer 5: Mesh Routing & Topology
* **Software Layer:** Protocol implementation.
* **Function:** Uses a Distance-Vector routing protocol (like LoRaMesher) combined with an exception-based dissemination strategy (only broadcasting anomalies or delta-breaches) to prevent mesh saturation.
* **Cost:** ₹0 (Open Source)

### Layer 6: Physical RF Modulation (LoRa PHY)
Translates MAC frames into RF chirps.

| Component | Function / Spec | Estimated Cost (INR) |
| :--- | :--- | :--- |
| **LoRa Transceiver** | SX1262 SPI Breakout Module. | ₹650 |
| **Antenna** | 865-867 MHz SMA Antenna (India ISM Band). | ₹200 |
| **Total Layer 6 Budget** | | **₹850** |

---
## Total Estimated Budget
* **Per Node Estimated Cost:** **₹7,690** (approx. $92.50 USD)