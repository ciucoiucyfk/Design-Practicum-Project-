# Project Overview: Decentralized Edge-Intelligent WSN

## 1. System Concept
This project aims to engineer a highly resilient, fully decentralized Wireless Sensor Network (WSN) designed for macro-regional deployment. By combining long-range RF communications (LoRa), environmental sensor arrays, and embedded machine learning (TinyML), the network generates high-resolution micro-climate maps and severe weather alerts without relying on a centralized cellular network or a single-point-of-failure gateway.

## 2. Core Objectives
*   **Decentralized Mesh Architecture:** Transition away from traditional LoRaWAN Star topologies. Nodes act autonomously as both data gatherers and multi-hop routers using Distance-Vector routing.
*   **Intelligence at the Edge:** Shift compute from the cloud to the silicon. Using quantized decision trees (TinyML), nodes locally classify raw data (e.g., detecting lightning patterns or VOC gas spikes) rather than transmitting raw telemetry.
*   **Exception-Based Broadcasting:** Solve the LoRa bandwidth bottleneck by remaining entirely silent unless a micro-climate anomaly threshold is broken.
*   **Absolute Autonomy:** Achieve multi-year field deployments using ultra-low-power deep sleep cycles, hardware peripheral switching, and localized MPPT solar energy harvesting.

## 3. Key Benefits & Real-World Impact
*   **Infrastructure Independence:** Because the network is completely peer-to-peer, it continues to function during cellular outages, severe storms, or grid failures. Any node can act as an access point to query the regional mesh.
*   **Hyper-Local Weather & Hazard Tracking:** Standard weather stations are sparse. This high-density grid can track the precise path of a microburst, a radiological plume, or a localized forest fire by correlating wind vectors, pressure drops, and VOCs across adjacent nodes.
*   **Extreme Bandwidth Efficiency:** By compiling intelligence down to the metal and utilizing SPIN-based negotiation handshakes, network saturation is avoided, allowing hundreds of nodes to share the India 865-867 MHz ISM band legally and efficiently.
*   **Secure by Design:** Every payload is authenticated and encrypted using hardware-accelerated AES and HMAC, ensuring that the sensor grid cannot be spoofed, hijacked, or mapped by bad actors.
