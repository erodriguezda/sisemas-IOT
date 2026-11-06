# Design & Decision Record (DDR)

**GreenField Technologies | IoT Systems Design**

**Team Members:**

* María de los Ángeles Prieto Ortega

* Mariana Zuluaga Yepes

## 1. System Overview

* **System Type:** \[x\] Component (Lab 1-2) | \[ \] System (Lab 3-6) | \[ \] Environment (Lab 7-8)

* **Board Identification (`ot extaddr`):**

  * **Node A (Leader):** `f4:ce:36:12:34:56:78:9a` *(Reemplazar con la dirección real obtenida en el laboratorio)*

  * **Node B (Intermediate Router):** `f4:ce:36:12:34:56:78:9b` *(Reemplazar con la dirección real obtenida en el laboratorio)*

  * **Node C (Final Router):** `f4:ce:36:12:34:56:78:9c` *(Reemplazar con la dirección real obtenida en el laboratorio)*

* **Description:**
  This Design & Decision Record documents the implementation and experimental results for **Lab 1 (RF Characterization)** and **Lab 2 (6LoWPAN Mesh Networking & Resilience)** using ESP32-C6 SoCs operating on IEEE 802.15.4 in the $2.4 \text{ GHz}$ band with OpenThread.

## 2. Lab Log & Stakeholder Summaries

### LAB 1 — RF Characterization

#### To Samuel (Architect)

The radio behavior of the ESP32-C6 using OpenThread was empirically validated. The cleanest channel found was channel 23, with an energy level of $-108 \text{ dBm}$ during the scan.

The link was completely stable up to $10 \text{ m}$, with $0\%$ packet loss ($\text{PER} = 0\%$) and an average RSSI of $-78 \text{ dBm}$. At $20 \text{ m}$, communication was still achieved, but packet loss increased to $20\%$. From $25 \text{ m}$ onwards, packet loss was high, between $31\%$ and $52\%$, meaning the link can no longer be considered reliable for a sensor network requiring high availability.

Conclusion: the ESP32-C6 works correctly for short-range Thread communication, but a practical maximum spacing of around $10 \text{ m}$ is recommended to ensure reliability under test conditions.

#### To Gustavo (Product Owner)

Based on our empirical field testing, the maximum reliable range between two ESP32-C6 nodes achieving $< 1\%$ Packet Error Rate ($\text{PER}$) is **10 meters** in indoor/semi-controlled environments.

**10-Hectare Field Deployment Calculation:**

* **Field Area:** $10 \text{ hectares} = 100,000 \text{ m}^2$.

* **Grid Coverage:** Using a grid spacing of $10 \text{ m}$ per node, each node covers an effective area of $10 \text{ m} \times 10 \text{ m} = 100 \text{ m}^2$.

* **Required Nodes:** $100,000 \text{ m}^2 / 100 \text{ m}^2 = \mathbf{1,000 \text{ nodes}}$.

* **Estimated Hardware Cost:** $1,000 \text{ nodes} \times \$40 \text{ USD/node} = \mathbf{\$40,000 \text{ USD}}$.

**Verdict: TEST MORE.**
We recommend proceeding to outdoor Line-of-Sight (LoS) testing before locking in the final architecture. If outdoor range extends to $25\text{--}30 \text{ m}$, the required node count will drop to $\approx 110\text{--}160 \text{ nodes}$, reducing hardware cost to $\$4,400\text{--}\$6,400 \text{ USD}$.

#### To Edwin (Ops)

The network operated correctly when both nodes used channel 23. If a node does not join or respond, verify that the `dataset channel` matches on both devices.

As distance increases, `NoAck` errors appear, meaning a frame was sent but no MAC ACK was received due to weak signal, obstacles, or excessive distance.

Recommendation: keep nodes at a maximum distance of $10 \text{ m}$ for reliable testing.

### LAB 2 — 6LoWPAN Mesh Networking & Resilience

#### To Samuel (Architect)

A 3-node multi-hop Thread mesh network was configured ($A \rightarrow B \rightarrow C$). Forced multi-hop routing was implemented using MAC filtering on Node A and Node C.

Key metrics:

* **1-hop latency (**$A \rightarrow B$**):** Average RTT of $34.9 \text{ ms}$.

* **2-hop latency (**$A \rightarrow C$ **via** $B$**):** Average RTT of $102.4 \text{ ms}$. Extra hop cost is $\approx 67.5 \text{ ms}$.

* **Address Standard:** Firmware applications must communicate using the **ML-EID** (Mesh Local EID) because RLOC addresses change dynamically when topology changes or nodes re-attach.

#### To Edwin (Ops)

The "Tractor Test" was conducted by abruptly disconnecting intermediate Router B while A was actively pinging C.

* **Measured Healing Time:** **64 seconds** (Average across 3 runs: $64 \text{ s}$, $58 \text{ s}$, $62 \text{ s}$).

* **Target:** Meets the requirement of $< 120 \text{ s}$ (**PASS**).

* **Field Procedure when a Node Goes Dark:**

  1. Check if neighboring nodes re-routed traffic automatically via `ot router table`.

  2. Verify if the dropped device was acting as the Leader or a solo Router.

  3. Ensure replacement nodes share the same active network dataset (`ot dataset active`).

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selection of Channel 23 for the Thread Network

* **Context:** The Thread network operates over IEEE 802.15.4 in the $2.4 \text{ GHz}$ band, overlapping Wi-Fi channels 1, 6, and 11.

* **Decision:** Channel 23 was selected after an energy scan (`ot scan energy 500`).

* **Rationale:** Channel 23 presented the lowest detected interference level at $-108 \text{ dBm}$. Channels 12–14 showed severe Wi-Fi Channel 1 interference (up to $-74 \text{ dBm}$).

* **Status:** \[x\] Accepted

### ADR-002: Choosing Thread Mesh Instead of LoRaWAN Star Topology

* **Context:** Farm deployment requires low latency for actuator control, localized resilience, and self-healing when farm equipment damages a routing node.

* **Decision:** Adopt Thread 6LoWPAN mesh topology instead of a LoRaWAN star topology.

* **Rationale:**

  * LoRaWAN uses a single-point-of-failure star topology; losing the central gateway drops all coverage.

  * Thread mesh provides autonomous dynamic routing around dead nodes with an observed healing time of $\approx 64 \text{ s}$ ($< 120 \text{ s}$).

  * Thread natively supports IPv6/CoAP end-to-end communication down to individual leaf nodes.

* **Status:** \[x\] Accepted

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification | Lab Introduced | 
| ----- | ----- | ----- | ----- | 
| ESP32-C6 SoC | SCD | Sensing/Controlling device | Lab 1 | 
| 802.15.4 Radio | SCD | Communication subsystem | Lab 1 | 
| Antenna | SCD | Physical interface | Lab 1 | 
| Air (RF medium) | PED | Electromagnetic propagation | Lab 1 | 
| Leader Node (A) | SCD | Coordinates mesh topology & dataset | Lab 2 | 
| Router Node (B) | SCD | Performs 6LoWPAN packet forwarding | Lab 2 | 
| IPv6 ML-EID | ASD Enabler | Application-layer reachability | Lab 2 | 

### IPv6 Addressing Classification (Task C)

| Address Type | Scope | Example / Format | Purpose / Usage | 
| ----- | ----- | ----- | ----- | 
| **Link-Local** | Interface / Link | `fe80::...` | Single-hop MAC communication, neighbor discovery, MLE messages. | 
| **RLOC (Routing Locator)** | Thread Mesh | `fd16:a9da:52bb:2755:0:ff:fe00:1400` | Used internally by Thread stack for efficient mesh routing. | 
| **ALOC (Anycast Locator)** | Thread Mesh | `fd16:a9da:52bb:2755:0:ff:fe00:fc00` | Addresses specific network roles (e.g., Leader ALOC). | 
| **ML-EID (Mesh Local EID)** | Thread Mesh | `fd16:a9da:52bb:2755:0:ff:fe00:1401` | **Application addressing.** Static endpoint IP that does not change upon topology shift. | 

#### Manual Address Derivations:

1. **Link-Local Interface ID:** Derived from Extended MAC address (`ot extaddr`). The **Universal/Local (U/L) bit** (Bit 1 of Byte 0) is inverted.

2. **RLOC16 Derivation:** The top 6 bits of the RLOC16 16-bit field represent the **Router ID** (matching the Router ID listed in `ot router table`), while the lower 10 bits represent the Child ID ($0$ for Routers).

#### Multicast Behavior:

* When a node transitions from **Router** to **Child**, it unsubscribes from the `ff03::2` (All-Routers multicast group), reducing broadcast traffic and power consumption.

### Component Capabilities & Thread Roles (Functional Viewpoint)

| Role | Functional Description | Power Profile | Impact on Battery | 
| ----- | ----- | ----- | ----- | 
| **Leader** | Manages router ID assignment and dataset. | Receiver Always On (RxOnWhenIdle) | High power consumption; requires mains/solar power. | 
| **Router** | Forwards mesh frames, maintains routing tables. | Receiver Always On (RxOnWhenIdle) | High power consumption; incompatible with 3-month battery target without solar. | 
| **REED** | Router-Eligible End Device; candidate router. | Receiver Always On (RxOnWhenIdle) | Medium-High power. | 
| **Sleepy End Device (SED)** | Sensor leaf node. | Duty-cycled / Sleep mode | **Low power; enables >3-month battery life.** | 

## 5. First Principles Reflections

### LAB 1

1. **Why does signal weaken with distance?**
   Follows Friis free-space path loss: $\text{FSPL (dB)} = 20 \log_{10}(d) + 20 \log_{10}(f) + 32.44$. Doubling distance drops power by $6 \text{ dB}$.

2. **Why is a stronger signal needed for** $\text{PER} < 1\%$ **than receiver sensitivity?**
   Requires a **Fade Margin** ($15\text{--}20 \text{ dB}$) above the noise floor to handle thermal noise spikes, multipath fading, and Wi-Fi interference bursts.

### LAB 2

1. **How does a 40-byte IPv6 header fit into a 127-byte frame?**
   Through **6LoWPAN Header Compression (RFC 6282)**. The 6LoWPAN stack elides prefix and Interface ID bits that match MAC layer source/destination addresses.

   * **Reconstruction:** The receiver rebuilds the **Link-Local** Interface ID directly from the 802.15.4 MAC source address (EUI-64). The **ML-EID** cannot be fully derived from the MAC header and requires explicit compression bits or mesh header fields.

2. **Why did healing take seconds instead of minutes?**
   Thread uses **MLE (Mesh Link Establishment)** link probes and Advertisement messages sent every few seconds. When Node A misses consecutive ACK/advertisements from B, MAC retries fail and the path cost is updated immediately. On an idle link with no traffic, node loss detection relies solely on periodic MLE advertisement timeouts ($\approx 32\text{--}64 \text{ s}$).

3. **Why does Task E require a *poor* A–C link instead of *no* link?**
   A poor direct link allows Node A to maintain Router C in its topology table with high Path Cost / Low Link Quality (LQ 1). When intermediate Router B is removed, A can immediately route directly to C over the degraded link without needing to trigger a full network rediscovery.

## 6. Performance Baselines & Experimental Results

### RF Energy Scan & Wi-Fi Overlap (Lab 1)

| 802.15.4 Channel | Wi-Fi Overlap | Measured RSSI \[dBm\] | Status / Notes | 
| ----- | ----- | ----- | ----- | 
| 11--14 | Wi-Fi Ch 1 | $-74 \text{ to } -93$ | High Interference | 
| 15 | Gap | $-101$ | Quiet | 
| 16--19 | Wi-Fi Ch 6 | $-83 \text{ to } -100$ | Moderate Interference | 
| 20 | Gap | $-101$ | Quiet Guard Band | 
| 21--24 | Wi-Fi Ch 11 | $-93 \text{ to } -108$ | **Ch 23 Cleanest (-108 dBm)** | 

### Multi-Hop RTT & Packet Loss (Lab 2 - Task D)

| Path | Hops | Min RTT | Avg RTT | Max RTT | Loss (PER) | Notes / MAC Frame Counters | 
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | 
| $A \rightarrow B$ | 1 | $15 \text{ ms}$ | $34.9 \text{ ms}$ | $53 \text{ ms}$ | $0\%$ | Direct link | 
| $A \rightarrow C$ (via B) | 2 | $48 \text{ ms}$ | $102.4 \text{ ms}$ | $210 \text{ ms}$ | $0\%$ | Forced via MAC filter. B Tx frame counter incremented by 20. | 
| Lab 1 ($A \rightarrow B$) | 1 | $15 \text{ ms}$ | $34.9 \text{ ms}$ | $53 \text{ ms}$ | $0\%$ | Baseline comparison | 

### Tractor Test Self-Healing Time (Lab 2 - Task E)

| Run | Pings Lost | Healing Time \[s\] | Route After Failure | Target Status | 
| ----- | ----- | ----- | ----- | ----- | 
| 1 | 64 | $64 \text{ s}$ | $A \rightarrow C$ (Direct degraded link) | Pass (< 120 s) | 
| 2 | 58 | $58 \text{ s}$ | $A \rightarrow C$ (Direct degraded link) | Pass (< 120 s) | 
| 3 | 62 | $62 \text{ s}$ | $A \rightarrow C$ (Direct degraded link) | Pass (< 120 s) | 
| **Average** | **61.3** | **61.3 s** | **Self-Healed** | **PASS** | 

## 7. Ethics & Sustainability Checklist

* \[x\] **Lab 1:** Verified passive scanning was used prior to transmission.

* \[x\] **Lab 2:** Mesh resilience tests did not disrupt adjacent group networks (used isolated Network Datasets and MAC filtering).

* \[x\] Radios disabled/off when experiments were completed to conserve power and reduce 2.4 GHz spectrum noise.