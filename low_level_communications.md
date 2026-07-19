# Layer 1: Low-Level Communications & Hardware Abstraction Layer (HAL)

This layer is strictly responsible for interacting with the physical hardware. It abstracts away the registers and protocols. **Crucially, this layer does not know what the data means, only how to retrieve it.**

## Modular Interface Enforcement
To guarantee strict modularity, Layer 1 must adhere to the following interface:
*   **Allowed Instructions:** Direct I2C/SPI register reads, GPIO interrupt handling, ADC sampling.
*   **Forbidden Instructions:** Network transmission, data scaling, machine learning math, AES encryption.
*   **Output Structure:** Layer 1 bundles raw readings into a C++ struct and pushes it to Layer 2 via a FreeRTOS Queue.

```cpp
// Explicit Output Interface to Layer 2
struct SensorDataStruct {
    float raw_temp;
    float raw_pressure;
    uint32_t voc_resistance;
    uint16_t wind_rpm;
    uint8_t rain_ticks;
};
// Sent via: xQueueSend(Queue_L1_to_L2, &data, portMAX_DELAY);
```

## 1. I2C Bus Management (BME680, AS3935)
The ESP32-S3 has two I2C controllers. To minimize wire routing and power consumption, multiple I2C devices will share a single bus.
*   **Bus Contention:** Since FreeRTOS is multi-threaded, accessing the I2C bus concurrently from different sensor tasks will cause crashes. You must use a `SemaphoreHandle_t i2c_mutex = xSemaphoreCreateMutex();`. 
*   **BME680:** Requires forced mode polling. The ESP32 wakes the BME680, triggers a measurement, waits ~150ms, reads the registers, and puts it to sleep.

## 2. SPI Bus Transactions (LoRa PHY - SX1262)
*(Note: Handled by Layer 6, but shares the hardware bus).*
*   **DMA (Direct Memory Access):** For sending LoRa packets, configure the ESP32 SPI driver to use DMA. 
*   **Chip Select (CS) Control:** Manage the CS pin explicitly before and after SPI transactions.

## 3. UART & Hardware FIFOs (NEO-6M GPS)
*   **Interrupt-Driven RX:** Configure a UART RX interrupt that pushes incoming bytes into a FreeRTOS Ring Buffer.
*   **Power Control:** The GPS is controlled by an N-Channel MOSFET. Once a fix is acquired, power it down entirely.

## 4. GPIO Interrupts & ADC (Wind, Rain, Radiation, Battery)
*   **Tipping Bucket (Rain):** Use a hardware RC low-pass filter to debounce, combined with an ESP32 ISR (`GPIO_INTR_NEGEDGE`).
*   **Rotary Encoder (Wind Speed):** Use the PCNT (Pulse Counter) hardware peripheral of the ESP32-S3 to count high-frequency pulses without CPU overhead.
*   **PIN Diode (Radiation):** Route the analog pulse to a peak-detector circuit and sample via the ADC on a rising edge trigger.
