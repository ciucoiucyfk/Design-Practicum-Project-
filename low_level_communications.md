# Layer 1: Low-Level Communications & Hardware Abstraction Layer (HAL)

This layer is strictly responsible for interacting with the physical hardware. It abstracts away the registers and protocols, providing clean data buffers to Layer 2 (Preprocessing) via FreeRTOS Queues. **Crucially, this layer does not know what the data means, only how to retrieve it.**

## 1. I2C Bus Management (BME680, AS3935)
The ESP32-S3 has two I2C controllers. To minimize wire routing and power consumption, multiple I2C devices will share a single bus.
*   **Bus Contention:** Since FreeRTOS is multi-threaded, accessing the I2C bus concurrently from different sensor tasks will cause crashes. You must use a `SemaphoreHandle_t i2c_mutex = xSemaphoreCreateMutex();`. Every I2C read/write must take the mutex, perform the transaction, and release it.
*   **BME680 (Address `0x76` or `0x77`):** Requires forced mode polling. Instead of continuous measurement, the ESP32 wakes the BME680, triggers a measurement, waits ~150ms, reads the registers (Temp, Hum, Press, Gas), and puts it back to sleep.
*   **AS3935 Lightning (Address `0x03` if I2C used):** Uses an interrupt pin (IRQ) for strikes, but requires I2C/SPI to read the strike distance and energy registers.

## 2. SPI Bus Transactions (LoRa PHY - SX1262/RFM95)
SPI is reserved for high-speed, crucial peripherals—primarily the LoRa radio.
*   **DMA (Direct Memory Access):** For sending and receiving LoRa packets, configure the ESP32 SPI driver to use DMA. This allows the CPU to sleep or process other tasks while the hardware shifts out the payload buffer.
*   **Chip Select (CS) Control:** Manage the CS pin explicitly before and after SPI transactions.
*   **Clock Speed:** LoRa modules typically handle 8-10 MHz SPI clocks comfortably. Keep traces short to avoid ringing.

## 3. UART & Hardware FIFOs (NEO-6M GPS)
GPS modules are notoriously power-hungry and slow to establish a fix.
*   **UART Configuration:** Set to 9600 baud (standard for NEO-6M).
*   **Interrupt-Driven RX:** Do not use blocking `while(Serial.available())` loops. Configure a UART RX interrupt that pushes incoming bytes into a FreeRTOS Ring Buffer.
*   **Power Control:** The GPS should NOT be powered constantly. A GPIO-controlled N-Channel MOSFET must switch the GPS power. Once a fix is acquired (or once every 24 hours to update the ephemeris), power it down entirely.

## 4. GPIO Interrupts & ADC (Wind, Rain, Radiation, Battery)
Interrupts are the backbone of a low-power sparse-traffic node. The CPU sleeps while the environment drives the logic.
*   **Tipping Bucket (Rain):** Generates a simple switch closure. 
    *   **Challenge:** Switch bounce. 
    *   **Solution:** Use a hardware RC low-pass filter (e.g., 10k resistor, 100nF capacitor) to debounce, combined with an ESP32 ISR configured for `GPIO_INTR_NEGEDGE`. The ISR increments a volatile `rain_ticks` counter.
*   **Rotary Encoder (Wind Speed):** High-frequency pulses.
    *   **Solution:** Attach an interrupt. To calculate RPM, either count pulses over a fixed timer (using an ESP32 hardware timer interrupt) or measure the time delta between pulses. For very high wind speeds, the PCNT (Pulse Counter) hardware peripheral of the ESP32-S3 is highly recommended over standard GPIO interrupts to save CPU cycles.
*   **PIN Diode (Radiation):** Outputs analog pulses proportional to particle energy, or simple digital counts. If digital, use the PCNT module. If analog, route the pulse to a peak-detector circuit and sample via the ADC, or trigger a high-speed ADC read on a rising edge.
*   **Battery Voltage (ADC):** Use a voltage divider (e.g., 100k/100k) to step the 4.2V max battery down to the 3.3V ESP32 ADC range. Since constant voltage dividers leak current, wire the bottom of the divider to a GPIO pin instead of ground. Drive the GPIO LOW only when taking a reading, otherwise set it to high-impedance (INPUT).

## ISR Safety Constraints
*   Inside any Interrupt Service Routine (ISR), you CANNOT use blocking code, `delay()`, or standard RTOS calls. 
*   You must use `...FromISR()` variants of FreeRTOS functions, like `xQueueSendFromISR()`, to pass a flag to Layer 2 that a sensor event occurred.
