# Compute Architecture: RTOS & Dual-Core Task Management

A primary concern in edge-intelligent sensor networks is "event blockage"—the risk that the CPU might be busy crunching a machine learning inference while a critical LoRa routing packet arrives, causing the packet to be dropped. 

To eliminate this bottleneck without stepping up to a power-hungry Linux Single Board Computer (SBC), this network leverages the **ESP32-S3 Dual-Core Processor** running a Real-Time Operating System (**FreeRTOS**).

## 1. The Power of Dual-Core Asymmetric Processing
The ESP32-S3 features two physical Xtensa LX7 cores running at up to 240MHz. Rather than letting the OS randomly assign tasks, we strictly "pin" specific architectural layers to specific cores to guarantee that time-sensitive network operations are never blocked by sensor polling or math calculations.

### Core 0 (The Protocol Core)
This core is dedicated entirely to keeping the node connected to the mesh. It handles all radio and cryptographic operations.
*   **Layer 6 (LoRa PHY / SPI DMA):** Manages the SX1262 LoRa module. Because it uses Direct Memory Access (DMA), the hardware handles incoming/outgoing bytes while the CPU handles routing logic.
*   **Layer 5 (Mesh Routing):** Maintains the Distance-Vector routing table and handles SPIN (ADV-REQ-DATA) negotiations.
*   **Layer 4 (Cryptography):** Executes hardware-accelerated AES-128-CTR and HMAC-SHA256 for packet authentication.

### Core 1 (The Application Core)
This core is dedicated entirely to local intelligence. If this core blocks for 150ms waiting for an I2C sensor, it has zero impact on Core 0's ability to route mesh traffic.
*   **Layer 1 (HAL / Sensor Polling):** Handles I2C reads, UART parsing for the GPS, and GPIO debouncing.
*   **Layer 2 (Preprocessing):** Maintains rolling windows and scales data into 1D Tensors.
*   **Layer 3 (TinyML Inference):** Executes the quantized decision tree (XGBoost/CatBoost) using ESP-NN vector instructions.

## 2. Preemptive Task Scheduling
Because we use FreeRTOS, the software does not run in a single blocking `loop()`. FreeRTOS uses *preemptive scheduling*.
*   If Core 1 is calculating an inference and an external interrupt fires (e.g., a lightning strike detected by the AS3935), FreeRTOS instantly preempts the inference task, executes the high-priority Interrupt Service Routine (ISR), and then seamlessly resumes the inference exactly where it left off.

## 3. Inter-Process Communication (IPC)
Because the layers are pinned to different cores, they cannot simply call each other's functions. They communicate exclusively via **FreeRTOS Queues**.
*   When Core 1 finishes an inference and detects a micro-climate anomaly, it pushes an 8-bit `AlertEvent` into a cross-core queue. 
*   Core 0, which is passively monitoring that queue, immediately wakes up, wraps the alert in AES encryption, and broadcasts it over the LoRa mesh. 

This decoupled, dual-core architecture provides the non-blocking performance of a distributed compute system while maintaining the micro-amp deep sleep capabilities required for long-term solar autonomy.
