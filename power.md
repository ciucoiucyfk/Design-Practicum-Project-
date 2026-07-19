# Layer 0: Power Distribution & Management

Because the decentralized mesh requires nodes to occasionally act as routers (listening and forwarding packets for neighbors), standard LoRaWAN "sleep for 24 hours" strategies do not apply. Power management must be highly aggressive and hardware-enforced.

## 1. Primary Power Grid
*   **Source:** 10W Solar Panel + 10,000mAh Lithium-Ion battery array.
*   **Controller:** An MPPT (Maximum Power Point Tracking) charge controller (e.g., CN3791 or similar) is required to efficiently harvest solar energy, especially during cloudy conditions.
*   **Voltage Regulation:** The ESP32 and sensors require a stable 3.3V. Use a highly efficient buck-boost converter or a low-quiescent-current (Iq) LDO (like the HT7333 or AP2112K) to drop the 3.7V-4.2V battery voltage to 3.3V.

## 2. Deep Sleep & Wake Stubs
The ESP32-S3 CPU consumes ~100mA when active. It must spend 99% of its life in `esp_deep_sleep()`, where it consumes less than 10µA.
*   **ULP Coprocessor:** The ESP32-S3 features a low-power RISC-V coprocessor. The ULP can remain awake, polling I2C sensors (like the BME680) or monitoring ADC pins, while the main CPU is powered down.
*   **EXT0/EXT1 Wakeups:** Wire the Tipping Bucket (Rain) and AS3935 (Lightning IRQ) to RTC GPIO pins. When a raindrop falls or lightning strikes, the hardware trigger instantly wakes the main CPU from deep sleep to record the event and evaluate if an alert needs to be sent.
*   **Timer Wakeup:** The node wakes up automatically every N minutes (e.g., 5 mins) to run the TinyML inference on the accumulated data buffer.

## 3. Hardware Peripheral Power Switching (MOSFETs)
Sensors consume power even when idle. To achieve true micro-amp standby:
*   Use logic-level N-Channel or P-Channel MOSFETs as high-side/low-side switches for power-hungry peripherals.
*   **GPS (NEO-6M):** GPS modules draw 30-50mA constantly. Connect the GPS VCC to a MOSFET controlled by an ESP32 GPIO. The node turns on the GPS upon boot to get a location fix, saves the coordinates to Non-Volatile Storage (NVS), and completely cuts physical power to the GPS until a reboot or manual request.
*   **BME680 & AS3935:** Can either be switched via MOSFET or placed into their respective hardware sleep modes (which draw ~1-5µA).

## 4. FreeRTOS Tickless Idle
When the CPU is awake but waiting for a sensor to respond (e.g., waiting 150ms for the BME680 forced measurement), do not use busy loops or `delay()`.
*   Configure FreeRTOS with `configUSE_TICKLESS_IDLE 1`.
*   When using `vTaskDelay()`, FreeRTOS will automatically put the CPU into Light Sleep (dropping current to ~1mA) until the exact microsecond the timer expires or an interrupt fires.
*   This ensures that even during active inference and data gathering, energy is conserved between CPU cycles.

## 5. Battery Monitoring & Failsafes
*   The ESP32 monitors the battery via an ADC voltage divider (switched on/off via GPIO to prevent divider current leak).
*   **Brownout Protection:** If the battery drops below 10% (e.g., ~3.2V), the node enters a "Survival Mode".
    *   It stops routing packets for other nodes.
    *   It stops all local sensor polling except the most critical (e.g., radiation).
    *   It only transmits a localized "Node Dying" beacon to allow neighbors to update their routing tables to route *around* it.
