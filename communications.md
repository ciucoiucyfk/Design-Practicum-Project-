# Layer 5 & 6: Mesh Routing, Node Integration, and Bandwidth Management

This document covers the multi-hop routing strategy, how new nodes dynamically join the mesh, and how the network strictly manages LoRa bandwidth.

## 1. Mesh Routing Topology (Distance-Vector vs. Reactive)
Standard LoRaWAN uses a centralized Star topology requiring a gateway. For a decentralized, macro-regional WSN, we use **Ad-Hoc Peer-to-Peer Mesh Routing**.

*   **Distance-Vector (DV) Routing:** Every node maintains a routing table of known destinations and the "next hop" to get there, along with the metric (hop count).
    *   *Implementation:* Similar to the `LoRaMesher` library for ESP32. Nodes periodically broadcast lightweight `HELLO` packets. 
    *   *Advantage:* Proactive. When a node needs to send an alert, the route is already known.
*   **SPIN (Sensor Protocol for Information via Negotiation):** To prevent redundant data from flooding the mesh (the "implosion" problem), we utilize a SPIN-like `ADV-REQ-DATA` handshake for propagating TinyML inference alerts.
    1.  **ADV (Advertise):** Node A detects a storm. It broadcasts a tiny ADV packet: `[MsgID: 12, Type: Storm]`.
    2.  **REQ (Request):** Node B receives it. If Node B hasn't seen `MsgID: 12` yet, it sends a REQ to Node A.
    3.  **DATA (Payload):** Node A sends the encrypted sensor payload to Node B. Node B then initiates its own ADV.

## 2. Auto-Discovery & Node Integration
When a new ESP32 node is powered on in the field, it must autonomously securely integrate into the existing mesh.

1.  **Passive Listening:** The new node enters LoRa RX mode on the predefined India ISM band (865-867 MHz) and listens for `HELLO` beacons from existing nodes.
2.  **HELLO Broadcast:** The new node broadcasts its own `HELLO` packet containing its MAC address and its current hop count to a designated sink (if one exists, otherwise it just registers as a peer).
3.  **Routing Table Update:** Neighboring nodes receive the `HELLO`, update their local DV routing tables, and rebroadcast a localized routing update.
4.  **Cryptographic Handshake:** 
    *   The network utilizes AES-128-CTR. All nodes must be pre-provisioned with the symmetric Network Key.
    *   To prevent replay attacks, the node requests the current rolling `nonce` or time-sync counter from its nearest neighbor. The neighbor authenticates the request via HMAC-SHA256 and provides the sync state.
5.  **Steady State:** The node calculates optimal routes based on RSSI (Signal Strength) and Hop Count, locking into the mesh.

## 3. Bandwidth Management & Airtime Optimization
LoRa physical constraints (duty cycles, time-on-air) mandate extreme payload compression.

### A. Exception-Based Broadcasting
Nodes remain entirely silent unless a critical event occurs. Routine telemetry is stored locally in flash memory or RAM. The radio only transmits when the TinyML engine outputs an anomaly (e.g., Output Index > 0), drastically reducing network congestion.

### B. Bit-Packed Payloads (C Structs)
Do not send JSON or strings over LoRa. Pack data into tightly constrained bit-fields.
```cpp
// Example: 6-byte packed payload
struct __attribute__((packed)) LoRaPayload {
    uint8_t  node_id;       // 0-255
    uint8_t  inference_idx: 4; // 0-15 classes
    uint8_t  confidence: 4;    // 0-15 scale
    int8_t   temp_delta;    // -128 to 127 Celsius offset
    uint16_t wind_rpm;      // 0-65535
    uint8_t  battery_pct;   // 0-100
};
```
### C. PHY Tuning (SF7/SF8)
*   The nodes are tuned specifically to Spreading Factors **SF7 and SF8**.
*   *Why?* Higher spreading factors (SF11/12) provide massive range but take seconds to transmit, saturating the channel and draining the battery. By using SF7/SF8, we enforce shorter, faster hops between densely packed mesh nodes rather than long, slow point-to-point links.
