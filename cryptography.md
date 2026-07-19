# Layer 4: Cryptographic Wrapper & Data Security

Because the network operates over a decentralized, multi-hop mesh on a public ISM band (865-867 MHz), every transmission must be encrypted and authenticated. Layer 4 sits between the TinyML Inference (Layer 3) and Mesh Routing (Layer 5). 

## 1. Cryptographic Suite
The node utilizes the ESP32-S3's built-in hardware cryptographic accelerators to ensure security without draining the battery via heavy CPU calculations.
*   **Encryption:** `AES-128-CTR` (Advanced Encryption Standard in Counter Mode).
*   **Authentication:** `HMAC-SHA256` (Hash-based Message Authentication Code).

## 2. Why AES-CTR Mode?
Most standard encryption (like AES-CBC) is a block cipher. It requires padding the data to a 16-byte boundary. If we have a highly optimized 6-byte payload (as defined in `communications.md`), padding it to 16 bytes wastes 10 bytes of precious LoRa airtime.
*   **Stream Cipher Advantage:** AES-CTR turns the block cipher into a stream cipher. It encrypts the data byte-by-byte. A 6-byte plaintext payload becomes exactly a 6-byte ciphertext payload. 

## 3. Packet Structure & Authentication
To ensure the routing layer (Layer 5) can move the packet without decrypting the payload, the network headers remain plaintext, while the payload is encrypted. 

```
[ MAC Header (Plain) ] [ Payload (Encrypted) ] [ HMAC Tag (Plain) ]
```

*   **Integrity & Anti-Spoofing:** Before trusting a packet, a receiving node calculates the `HMAC-SHA256` hash of the Header + Encrypted Payload using a pre-shared Network Key. If the calculated hash matches the `HMAC Tag` attached to the packet, the node knows the data was not tampered with and truly came from a valid network peer.

## 4. Replay Attack Prevention
A bad actor could record an encrypted "Severe Storm Warning" packet and replay it days later to trigger false alarms.
*   **The Nonce:** Every AES-CTR operation requires a unique Initialization Vector (IV) or Nonce. 
*   **Rolling Counter:** The network uses a 32-bit synchronized counter. Every time a node sends a message, it increments its counter. This counter forms the nonce.
*   **Handshake Validation:** As outlined in `communications.md` (Auto-Discovery), when a new node joins, it requests the current network counter state. If a node receives a packet with a counter value lower than or equal to the last known value from that sender, the packet is instantly dropped as a replay attack.

## 5. Key Management
*   **Pre-Shared Key (PSK):** For Phase 1, a symmetric 128-bit Network Key is baked into the secure Non-Volatile Storage (NVS) of the ESP32 during flashing. 
*   **Future Scope (Diffie-Hellman):** For advanced deployments, nodes could utilize Elliptic Curve Diffie-Hellman (ECDH) key exchanges to negotiate unique session keys with their immediate neighbors, compartmentalizing the network so that one physically compromised node does not expose the entire macro-regional mesh.
