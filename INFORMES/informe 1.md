# Design & Decision Record (DDR)

**GreenField Technologies | IoT Systems Design**

**Team:** Edwin santiago Rodriguez Daza - Jhonatan Yara Lopez

## 1. System Overview

* **System Type:** \[x\] Component (Lab 1-2) | \[ \] System (Lab 3-6) | \[ \] Environment (Lab 7-8)

* **Board Identification (`ot extaddr`):**

  * **Node A (Leader):** `f4:ce:36:12:34:56:78:9a` *(Reemplazar con sus direcciones reales si difieren)*

  * **Node B (Router):** `f4:ce:36:12:34:56:78:9b` *(Reemplazar con sus direcciones reales si difieren)*

* **Description:**

An initial radiofrequency characterization of the ESP32-C6 was carried out using OpenThread. The objective was to validate whether two ESP32-C6 nodes can communicate reliably using IEEE 802.15.4 radio in the 2.4 GHz band. To achieve this, an energy scan per channel was performed, the channel with the lowest interference was selected, and transmission and reception tests were conducted using `ot ping` at different distances.

The ESP32-C6 was configured and debugged using JTAG, which allowed working with the ESP-IDF development environment and directly observing OpenThread messages during the tests.

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

#### To Samuel (Architect)

The radio behavior of the ESP32-C6 using OpenThread was empirically validated. The cleanest channel found was channel 23, with an energy level of -108 dBm during the scan. After configuring the network on this channel, IPv6 ping tests were performed at various distances.

The link was completely stable up to 10 m, with 0% packet loss (PER = 0%) and stable RSSI values around -78 dBm. At 20 m, communication was still achieved, but packet loss increased to 20%. From 25 m and 30 m onwards, packet loss was high, between 31% and 52%, meaning the link can no longer be considered reliable for a sensor network that requires high availability.

Conclusion: the ESP32-C6 works correctly for short-range Thread communication, but based on the obtained data, a practical maximum spacing of around 10 m is recommended to ensure reliability.

#### To Gustavo (Product Owner)

Based on our empirical field testing, the maximum reliable range between two ESP32-C6 nodes achieving < 1% Packet Error Rate (PER) is **10 meters** in indoor/semi-controlled environments.

**10-Hectare Field Deployment Calculation:**

* **Field Area:** 10 hectares = $100,000 \text{ m}^2$.

* **Grid Coverage:** Using a grid spacing of 10 meters per node, each node covers an effective area of approximately $10\text{ m} \times 10\text{ m} = 100 \text{ m}^2$.

* **Required Nodes:** $100,000 \text{ m}^2 / 100 \text{ m}^2 = \mathbf{1,000\text{ nodes}}$.

* **Estimated Hardware Cost:** $1,000 \text{ nodes} \times \$40\text{ USD/node} = \mathbf{\$40,000\text{ USD}}$.

**Verdict: TEST MORE.**
We recommend proceeding to outdoor Line-of-Sight (LoS) testing with canopy/vegetation simulation before locking in the final architecture. If outdoor range extends to 25–30 meters, the required node count will drop significantly to \~110–160 nodes, reducing hardware cost to \$4,400–$6,400 USD.

#### To Edwin (Ops)

The network operated correctly when both nodes used the same channel. The configured channel was 23. If a node does not join the network or does not respond to ping, the first thing to check is that the `dataset channel` matches on both devices.

It was also observed that as distance increases, `NoAck` errors appear, meaning that a packet was sent but no acknowledgment was received from the other node. This typically occurs due to weak signal, obstacles, antenna orientation, interference, or excessive distance.

Recommendation: keep the nodes at a maximum distance of 10 m for reliable testing in indoor environments or in environments with uncontrolled interference.

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selección del canal 23 para la red Thread

* **Context:**

The Thread network operates over IEEE 802.15.4 in the 2.4 GHz band. This band overlaps heavily with Wi-Fi channels 1, 6, and 11. Specifically, Wi-Fi channel 1 overlaps 802.15.4 channels 11–14; Wi-Fi channel 6 overlaps channels 16–19; and Wi-Fi channel 11 overlaps channels 21–24.

Before creating the network, an energy scan was performed using `ot scan energy 500` to find a channel located in the spectral guard gaps or experiencing minimal local Wi-Fi pollution.

* **Decision:**

Channel 23 was selected to operate the Thread network.

* **Rationale:**

Channel 23 showed the lowest detected energy level during the scan, at -108 dBm. Although channel 23 lies near the upper edge of Wi-Fi channel 11, the local environment had low traffic on that spectrum segment during testing. Channels 12, 13, and 14 had severe interference (-74 dBm to -81 dBm) due to an active local Wi-Fi Access Point on Wi-Fi Channel 1.

* **Status:** \[ \] Proposed | \[x\] Accepted | \[ \] Deprecated

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification | 
| ----- | ----- | ----- | 
| ESP32-C6 SoC | SCD | Sensing/Controlling device (Section 6.4-6.5 of standard) | 
| 802.15.4 Radio | SCD | Communication subsystem | 
| Antenna | SCD | Physical interface to PED | 
| Air (RF medium) | PED | Physical entity - electromagnetic propagation | 

### Component Capabilities (ISO/IEC 30141 §9 Tables 9–10)

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced | Reason / Justification | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| Transducer | Actuating | On-board LED | Latent | Lab 1 | Board hardware includes LED, but not driven in Lab 1. | 
| Processing | Embedded control | ESP32-C6 RISC-V Core | Active | Lab 1 | Runs Zephyr / OpenThread stack. | 
| Communication | RF transmission | 802.15.4 Radio (2.4 GHz) | Active | Lab 1 | Transmitting ping/ACK packets. | 
| Communication | Physical interface | Antenna | Active | Lab 1 | Converts electrical signals to RF waves. | 
| Data | Storage | Internal NVS Flash | Active | Lab 1 | Stores OpenThread network dataset. | 
| Data | Network stack | OpenThread Shell / IPv6 | Active | Lab 1 | Provides CLI interface & IPv6 routing. | 
| Supporting | Hardware Security | Crypto Accelerator (AES) | Latent | Lab 1 | Used under the hood by MAC layer, explicitly active in Lab 6. | 
| Other radios | Wireless | Wi-Fi / BLE | Latent | Lab 1 | Hardware supports Wi-Fi 6 & BLE 5, kept disabled to save power. | 
| Transfer | Propagation | Air (RF medium) | Latent (PED) | Lab 1 | Physical channel subject to noise and attenuation. | 

## 5. First Principles Reflections

### Lab 1

#### 1. Why does the signal weaken as distance increases?

A radio signal propagates as an electromagnetic wave. As it moves away from the transmitter, its energy spreads over a larger spherical surface area. In free space, this decay is governed by the Inverse-Square Law (Friis Transmission Equation): doubling the distance reduces received power by $6\text{ dB}$ (a factor of 4).

$$
\text{FSPL (dB)} = 20 \log_{10}(d) + 20 \log_{10}(f) + 32.44
$$

#### 2. The receiver decodes signals near -100 dBm. Why is a stronger signal needed for PER < 1%?

While receiver sensitivity allows decoding single packets near the hardware noise floor (approx. $-100\text{ dBm}$), real-world channels suffer from thermal noise, fading, multipath interference, and transient Wi-Fi bursts.

To guarantee a Packet Error Rate (PER) under 1%, the link requires an adequate **Fade Margin** (typically $15\text{ to }20\text{ dB}$ above sensitivity). Without this margin, minor environmental changes cause signal dips below receiver threshold, leading to dropped frames and `NoAck` errors.

#### 3. Why is IEEE 802.15.4 used instead of WiFi for sensors?

IEEE 802.15.4 is optimized for low duty-cycles, small payload sizes (127 bytes total frame), and low power consumption (operating on batteries for years). Wi-Fi provides megabit throughput but demands high peak power and complex connection handshakes, making it unsuitable for battery-operated agricultural sensor nodes.

#### 4. What does selecting the channel with the lowest RSSI mean?

In an energy scan (`ot scan energy`), the radio measures background energy across all channels without decoding frames. Lower (more negative) values mean cleaner spectrum with less ambient RF noise from coexisting Wi-Fi access points or Bluetooth devices.

## 6. RF Scan & Wi-Fi Overlap Results

### Energy Scan & Wi-Fi Channel Mapping

Command used: `ot scan energy 500`

| 802.15.4 Channel | Overlapping Wi-Fi Channel | Measured RSSI \[dBm\] | Status / Observations | 
| ----- | ----- | ----- | ----- | 
| 11 | Wi-Fi Ch 1 | \-93 | Moderate Wi-Fi traffic | 
| 12 | Wi-Fi Ch 1 | \-78 | High Wi-Fi interference | 
| 13 | Wi-Fi Ch 1 | \-74 | **Highest interference** (Wi-Fi AP center frequency) | 
| 14 | Wi-Fi Ch 1 | \-81 | High Wi-Fi interference | 
| 15 | Gap (Ch 1 - Ch 6) | \-101 | Relatively quiet | 
| 16 | Wi-Fi Ch 6 | \-97 | Low Wi-Fi traffic | 
| 17 | Wi-Fi Ch 6 | \-83 | Moderate Wi-Fi interference | 
| 18 | Wi-Fi Ch 6 | \-99 | Quiet | 
| 19 | Wi-Fi Ch 6 | \-100 | Quiet | 
| 20 | Gap (Ch 6 - Ch 11) | \-101 | Quiet guard band | 
| 21 | Wi-Fi Ch 11 | \-98 | Low traffic | 
| 22 | Wi-Fi Ch 11 | \-93 | Moderate traffic | 
| 23 | Wi-Fi Ch 11 | **-108** | **Cleanest Channel Selected** | 
| 24 | Wi-Fi Ch 11 | \-96 | Low traffic | 
| 25 | Guard Band / Ch 14 | \-102 | Quiet | 
| 26 | Upper Edge | \-96 | Low traffic | 

## 7. Range & RSSI Testing Results (Task D)

### Table D: RSSI and Packet Loss against Distance

* **Base Command:** `ot ping fd16:a9da:52bb:2755:0:ff:fe00:1400 64 100 0.2`

* **Neighbor Table Command:** `ot neighbor table`

| Distance (m) | RSSI A→B (dBm) | RSSI B→A (dBm) | PER A→B (%) | PER B→A (%) | RTT Avg (ms) | Status | 
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | 
| 1 m | \-42 | \-44 | 0% | 0% | 34.9 ms | Reliable (PER < 1%) | 
| 5 m | \-61 | \-63 | 0% | 0% | 38.1 ms | Reliable (PER < 1%) | 
| 10 m | \-76 | \-78 | 0% | 0% | 35.4 ms | **Max Reliable Threshold** | 
| 15 m | \-88 | \-90 | 39% | 37% | 281.5 ms | Unreliable (PER > 1%) | 
| 20 m | \-85 | \-87 | 20% | 22% | 97.1 ms | Unreliable | 
| 25 m | \-92 | \-93 | 31% | 33% | 220.3 ms | High Loss | 
| 30 m | \-96 | \-97 | 52% | 50% | 128.9 ms | Unreliable | 
| 35 m | \-98 | \-99 | 52% | 54% | 203.9 ms | Link Boundary | 

* **RSSI Threshold for PER < 1%:** Approximately **-78 dBm**.

* **Maximum Reliable Range:** **10 meters** under indoor test conditions.

## 8. Performance Baselines

| Metric | Target | Measured | Status | 
| ----- | ----- | ----- | ----- | 
| Lab 1: Max Range | \> 20 m | 10 m reliable (PER < 1%) / 20 m (20% loss) | \[ \] Pass / \[x\] Partial | 
| Lab 2: Healing Time | < 120 s | Pending | \[ \] Pending | 
| Lab 3: CoAP Latency | < 200 ms | Pending | \[ \] Pending | 
| Lab 4: Poll Latency | < 5 s | Pending | \[ \] Pending | 
| Lab 6: DTLS Time | < 3 s | Pending | \[ \] Pending | 

## 9. Ethics & Sustainability Checklist

* \[x\] **Lab 1:** Verified interference doesn't disrupt neighbors. Passive energy scanning was used prior to channel selection, avoiding active RF jamming.

* \[x\] **Lab 1:** Radio power management: radios are powered down when not actively executing link measurements.

## 10. Answers to Final Lab Questions

1. **What is the maximum reliable range between sensor nodes?**

   **10 meters** (achieving 0% packet loss and steady \~-78 dBm RSSI).

2. **Which channel gives the cleanest spectrum in the 2.4 GHz band?**

   **Channel 23** (-108 dBm noise floor).

3. **What RSSI threshold predicts 99% packet delivery (PER < 1%)?**

   An RSSI of **-78 dBm** or stronger predicts 99%+ packet delivery on the ESP32-C6 platform.

4. **Did the ESP32-C6 meet the expected range requirement?**

   Partially. The requirement was > 20 m, but reliable communication (< 1% PER) was capped at 10 m under test conditions.