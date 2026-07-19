# Layer 3: Edge Intelligence & TinyML Inference

The TinyML engine is the brain of the node, turning raw sensor telemetry into actionable, highly compressed classifications. Instead of sending 20 bytes of floating-point sensor data over LoRa, we send a 1-byte "classification index" and confidence score.

## 1. Preprocessing (Layer 2 to Layer 3 Handoff)
Before inference, Layer 2 must format the raw HAL data into a **1D Tensor** (an array of floats).
*   **Scaling:** ML models require normalized inputs. The ESP32 must apply the exact same `StandardScaler` or `MinMaxScaler` logic applied during Python training.
    *   *Example:* `scaled_temp = (raw_temp - mean_temp) / std_temp;`
*   **Feature Engineering (Rolling Windows):** A sudden drop in pressure is more important than absolute pressure. Layer 2 maintains a circular buffer of the last N readings to calculate deltas (e.g., $\Delta P$ over 30 mins) to feed into the tensor.

## 2. Model Architecture (XGBoost / CatBoost via emlearn/micromlgen)
For tabular environmental data, decision trees heavily outperform Neural Networks in terms of accuracy per CPU cycle.
*   **Quantization:** The trained model is converted into nested `if-else` C++ arrays. No heavy matrix multiplications (like in CNNs) are required. 
*   **Memory Footprint:** A highly accurate random forest with 50 trees and depth 5 can compile down to just 15-20 KB of Flash and use near-zero RAM.

## 3. Types of Inferencing Supported
By fusing the specific sensor array, the node can generate localized predictions:

### A. Micro-Climate Severe Weather Warning
*   **Inputs:** BME680 ($\Delta$ Pressure, $\Delta$ Temp, absolute humidity) + Wind RPM + AS3935 Lightning Distance.
*   **Inference:** Detects the signature of an incoming microburst, squall, or severe thunderstorm cell minutes before impact. 
*   **Output Index:** `0` (Clear), `1` (Rain likely), `2` (Severe Storm Imminent).

### B. Environmental Hazard / Wildfire Detection
*   **Inputs:** BME680 (VOC Gas Resistance, Temp, Humidity) + Wind Direction (if inferred or added).
*   **Inference:** Rapid drops in gas resistance coupled with rising temperature indicate smoke/fire proximity or a chemical/pollution dump.
*   **Output Index:** `0` (Normal AQI), `1` (Poor AQI), `2` (Fire/Smoke Hazard).

### C. Radiological Plume Tracking
*   **Inputs:** PIN Diode counts + Wind Speed.
*   **Inference:** Detects abnormal Beta/Gamma radiation spikes. Cross-referencing this locally with wind speed helps determine if the anomaly is static or moving across the macro-region.
*   **Output Index:** `0` (Background), `1` (Elevated), `2` (Critical Anomaly).

## 4. The "Exception-Based" Trigger
The inference engine acts as the gatekeeper to the LoRa radio. 
The node runs inference locally every 5 minutes. 
*   If Output Index == `0` (Normal): The node goes back to sleep. **Zero radio transmission.**
*   If Output Index > `0` (Anomaly): The node triggers Layer 4 (Crypto) and Layer 5 (Routing) to immediately broadcast an alert packet.
