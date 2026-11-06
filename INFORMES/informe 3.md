# Design & Decision Record (DDR)

**GreenField Technologies | IoT Systems Design**

**Team Members:**

* Edwin Santiago Rodriguez Daza 

* Jhonatan Yara Lopez

## 1. System Overview

* **System Type:** \[ \] Component (Lab 1-2) | \[x\] System (Lab 3-6) | \[ \] Environment (Lab 7-8)

* **Board Identification (`ot extaddr`):**

  * **Node A (Leader / Client):** `f4:ce:36:12:34:56:78:9a` *(Reemplazar con real)*

  * **Node B (Router):** `f4:ce:36:12:34:56:78:9b` *(Reemplazar con real)*

  * **Node C (SED / Sensor & Valve):** `f4:ce:36:12:34:56:78:9c` *(Reemplazar con real)*

* **Description:**
  This DDR documents the experimental results for **Lab 1 (RF Characterization)**, **Lab 2 (6LoWPAN Mesh)**, and **Lab 3 (CoAP/CBOR Application Transport)**. The system implements a Thread mesh using ESP32-C6 devices, providing a highly efficient uplink for sensor data (using CoAP Observe) and a reliable downlink for actuating sleepy valves.

## 2. Lab Log & Stakeholder Summaries

### LAB 1 & 2 — RF Characterization & Mesh Networking

#### To Samuel (Architect) & Gustavo (Product Owner)

The ESP32-C6 radio was validated on **Channel 23** (cleanest at $-108 \text{ dBm}$). The maximum reliable range ($\text{PER} < 1\%$) is $10 \text{ m}$ indoor. For a $10 \text{ hectare}$ field, a $10 \text{ m}$ grid requires $\approx 1,000 \text{ nodes}$ ($\approx \$40,000 \text{ USD}$). **Verdict:** Need outdoor LoS testing to increase range and reduce node count.
The network forms a resilient multi-hop mesh. The "Tractor Test" proved self-healing capabilities, taking $\approx 64 \text{ s}$ to route around a crushed node, successfully meeting the $< 120 \text{ s}$ requirement.

### LAB 3 — Efficient Data Transport (CoAP & CBOR)

#### To Daniela (Customer)

Readings now consume significantly less battery and arrive exactly when they change. CoAP/CBOR shrunk the radio message to just $\approx 36 \text{ B}$ (about $14\times$ smaller than HTTP). More importantly, the `Observe` feature means the sensor only transmits when the temperature actually changes (we measured $29$ updates in $5 \text{ min}$ compared to $343$ if the dashboard polled every second). This saves massive amounts of radio power, keeping your sensors alive much longer.

#### To Edwin (Ops)

When you command a valve to open/close from the dashboard, the system uses CoAP **CON (Confirmable)** requests. This guarantees the valve replies with an ACK. If the valve is crushed or offline, the gateway retries $4$ times and then "fails loudly" after a `ResponseTimeout` ($\approx 45 \text{ s}$), alerting you on the dashboard.
For battery-operated valves, expect a short delay. We configured the valves to sleep and wake up to check for commands every $5 \text{ s}$ (Poll Period). Therefore, downlink latency to a sleeping valve will be between $1 \text{ ms}$ and $5 \text{ s}$.

## 3. Architecture Decision Records (ADRs)

### ADR-001: Thread Channel 23

* **Decision:** Use Channel 23 ($-108 \text{ dBm}$) to avoid severe Wi-Fi Ch 1 interference. \[x\] Accepted

### ADR-002: Thread Mesh Topology

* **Decision:** Adopt Thread 6LoWPAN mesh instead of LoRaWAN star to enable local resilience and multi-hop routing around dead nodes (heals in $< 120 \text{ s}$). \[x\] Accepted

### ADR-003: CoAP/UDP + CBOR for Sensor Uplink

* **Decision:** Use CoAP/UDP with `Observe` and CBOR over MQTT/HTTP.

* **Rationale:** Reduces wire size to $\approx 36 \text{ B}$ and drops transmission frequency by $\approx 91\%$ compared to $1 \text{ Hz}$ polling. Avoids TCP handshake and keep-alive overhead. \[x\] Accepted

### ADR-004: 5-Second Poll Period for Sleepy Valves (SED)

* **Context:** Valves must run on batteries for months, requiring the ESP32-C6 radio to remain in Deep Sleep (SED role). However, the dashboard needs responsive actuation.

* **Decision:** Set the Sleepy End Device (SED) poll period to $5 \text{ s}$.

* **Rationale:** Based on Task E, a $5 \text{ s}$ poll period provides an acceptable maximum downlink latency of $5 \text{ s}$ while keeping the radio off $> 99\%$ of the time. A $1 \text{ s}$ poll drained the battery too fast, and $15 \text{ s}$ made the dashboard feel unresponsive.

* **Status:** \[x\] Accepted

## 4. ISO/IEC 30141 Mapping

### Domain Mapping & Component Capabilities

| Component / Capability | ISO Domain | Active / Latent | Lab | 
| ----- | ----- | ----- | ----- | 
| ESP32-C6 / 802.15.4 Radio | SCD | Active | Lab 1 | 
| IPv6 Mesh Routing / MLE | SCD | Active | Lab 2 | 
| `/env/temp` (CoAP Observe) | ASD | Active | Lab 3 | 
| `/act/valve` (CoAP CON) | ASD | Active | Lab 3 | 

### Application Service Contracts (Lab 3)

**1. Uplink: Temperature Sensor**

* **Resource:** `/env/temp` (Methods: `GET`, `Observe`)

* **Format:** `application/cbor` (Content-Format $60$)

* **CDDL:** `env-reading = { t: float16 }`

**2. Downlink: Valve Actuator (New)**

* **Resource:** `/act/valve` (Methods: `PUT`, `GET`)

* **Transport:** CoAP `CON` (Confirmable) to ensure delivery.

* **Format:** `application/cbor`

* **CDDL Payload:** `valve-cmd = { state: bool }` (e.g., `true` = Open)

* **Error Handling:** Returns `4.00 Bad Request` if payload is malformed.

## 5. First Principles Reflections (Lab 3 Additions)

**1. Why can CoAP use UDP where HTTP needs TCP? What do CON and NON do?**
CoAP handles its own reliability and message ID tracking natively over connectionless UDP.

* **CON (Confirmable):** Guarantees delivery by demanding an ACK from the receiver, retransmitting if lost.

* **NON (Non-confirmable):** Deliberately gives up ACKs and reliability to achieve the absolute minimum byte overhead (used for frequent sensor telemetry where losing one packet doesn't matter).

**2. Why does Observe save battery compared with a GET every 60 s? Who pays for the radio?**
In standard polling, the client forces the battery-powered sensor to turn on its TX/RX radio to answer "no change". With `Observe`, the **sensor** dictates when to transmit. The sensor only "pays" the radio energy cost when the physical value crosses the defined threshold.

**3. Why did the sleepy valve receive the same PUT twice, and why was that harmless?**
The CoAP client's default initial retransmission timeout ($2\text{--}3 \text{ s}$) was shorter than the SED's poll period ($5 \text{ s}$). The client assumed the first packet was lost and sent a retry just as the valve woke up, receiving both. This is harmless because the PUT payload (`{state: true}`) is **idempotent**—setting the valve to OPEN twice leaves it OPEN. If we had used a "toggle" command, the duplicate would have opened and immediately closed the valve, ruining the irrigation cycle.

**4. Why CBOR and not a hand-made binary struct?**
CBOR provides a standardized, self-describing binary format. Unlike raw C-structs, CBOR avoids endianness (byte-order) issues, padding inconsistencies, and allows different programming languages (e.g., a Python dashboard and a C firmware) to seamlessly exchange structured data using a CDDL schema.

## 6. Performance Baselines & Experimental Results

| Metric | Target | Measured | Status | 
| ----- | ----- | ----- | ----- | 
| **Lab 2: Healing Time** | $< 120 \text{ s}$ | $64 \text{ s}$ | \[x\] Pass | 
| **Lab 3: CoAP Latency (1 hop)** | $< 200 \text{ ms}$ | $44 \text{ ms}$ avg | \[x\] Pass | 
| **Lab 3: Poll Latency (5 s period)** | $< 5 \text{ s}$ | $\approx 2.5 \text{ s}$ avg | \[x\] Pass | 

### Task D - Downlink CON Timeout Validation

When sending a `CON PUT` to a node that has left the network, the client attempts $4$ retransmissions with exponential backoff.

* **Predicted Timeout:** RFC 7252 defines `MAX_TRANSMIT_SPAN` as $\approx 45 \text{ s}$.

* **Measured ResponseTimeout:** $46 \text{ s}$.

### Task E - Sleepy End Device (SED) Latency

| Poll Period | Min RTT | Avg RTT | Max RTT | Estimated Radio-On Fraction | 
| ----- | ----- | ----- | ----- | ----- | 
| **1 s** | $12 \text{ ms}$ | $490 \text{ ms}$ | $985 \text{ ms}$ | $\approx 1.0\%$ (drains battery quickly) | 
| **5 s** | $15 \text{ ms}$ | $2,450 \text{ ms}$ | $4,980 \text{ ms}$ | $\approx 0.2\%$ (excellent balance) | 
| **15 s** | $18 \text{ ms}$ | $7,500 \text{ ms}$ | $14,950 \text{ ms}$ | $\approx 0.06\%$ (too slow for dashboard) | 

## 7. Ethics & Sustainability Checklist

* \[x\] **Lab 1 & 2:** Passive scans used; radios disabled when not testing.

* \[x\] **Lab 3 (Sustainability):** Observe reduced radio transmissions by $\approx 91.55\%$. Sleepy End Device (SED) polling at $5 \text{ s}$ keeps the radio off $> 99\%$ of the time.

* \[x\] **Lab 3 (Safety / Failsafe):** What if the valve receives `OPEN` and then the network dies?

  * **Design requirement:** The valve firmware must implement an internal hardware timer (watchdog). If the valve is OPEN but receives no network heartbeat/commands for $15 \text{ minutes}$, it must **autonomously execute a CLOSE operation** to prevent flooding the farmer's field and wasting critical water resources.
