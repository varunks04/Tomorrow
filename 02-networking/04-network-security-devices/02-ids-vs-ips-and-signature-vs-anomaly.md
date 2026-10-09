# IDS vs. IPS, Detection Methodologies & Alert Tuning

> **Domain:** Networking Fundamentals & Network Security  
> **Sub-Domain:** Intrusion Detection & Prevention Systems  
> **Interview Importance:** Very High / Classic Security Operations Question  

---

## 1. Topic & Definitions

- **Intrusion Detection System (IDS):** A passive, out-of-band network security appliance that monitors network traffic for suspicious activity, known exploit signatures, or policy violations, generating security alerts for security analysts without actively altering or blocking traffic.
- **Intrusion Prevention System (IPS):** An active, in-line network security appliance that sits directly in the path of network traffic, inspecting packets in real-time and actively **blocking, dropping, or terminating** malicious sessions upon detection.
- **SPAN Port (Port Mirroring):** A software switch feature that copies network traffic from designated ports and forwards the duplicate mirror stream to an analysis port connected to an IDS.
- **Network TAP (Test Access Point):** A dedicated hardware device spliced directly into physical network cabling that splits physical light or electrical signals to send an exact duplicate packet stream to an out-of-band monitoring tool with zero latency impact.

---

## 2. Deployment Architecture: In-Line (IPS) vs. Out-of-Band (IDS)

```text
PASSIVE / OUT-OF-BAND DEPLOYMENT (IDS):
[ Router / Switch ] ──────────────────────────────────────────► [ Destination Server ]
        │ (Physical wire)
        ├─ [ SPAN / TAP Mirror Stream ] ──► [ IDS Appliance (e.g. Snort/Zeek) ]
        │                                           │
        │                                           ▼ Generates Alert / SIEM Log
        │                                     (Cannot stop malicious packet from reaching server!)

IN-LINE DEPLOYMENT (IPS):
[ Router / Switch ] ──► [ IPS Appliance ] ────────────────────► [ Destination Server ]
                             │
                             ├─ Evaluates packet in real time
                             ├─ [CLEAN]: Forwards packet to destination
                             └─ [MALICIOUS]: DROPS packet / Injects TCP RST! (Attack blocked)
```

---

## 3. Key Differences: IDS vs. IPS

| Dimension | IDS (Intrusion Detection System) | IPS (Intrusion Prevention System) |
| :--- | :--- | :--- |
| **Physical Placement** | **Out-of-band / Passive** (SPAN port / TAP). | **In-line** (Directly in traffic path). |
| **Action Taken** | Alerts only (Syslog, SNMP, SIEM event). | **Active intervention:** Drops packet, resets connection (TCP RST), blocks IP. |
| **Network Latency Impact**| **Zero latency** on production traffic. | Introduces minor packet processing latency (1-3 ms). |
| **Failure Mode** | **Fails Safe / Open:** If IDS crashes, live network traffic continues unaffected. | **Single Point of Failure:** If IPS crashes, it blocks traffic unless an active bypass switch is used. |
| **Risk of False Positives**| Creates nuisance alert noise in SOC. | **Severe:** Accidentally blocks legitimate business transactions or VIP users. |

---

## 4. Detection Methodologies Explained

```text
+─────────────────────────────────────────────────────────────────────────────+
|                         DETECTION METHODOLOGIES                             |
+─────────────────────────────────────────────────────────────────────────────+
| 1. SIGNATURE-BASED DETECTION (Knowledge-Based):                             |
|    • Compares traffic payloads against database of known attack patterns.  |
|    • Pros: High accuracy, low false positives on known CVEs.                |
|    • Cons: Completely blind to Zero-Day exploits and polymorphic malware.   |
|─────────────────────────────────────────────────────────────────────────────|
| 2. ANOMALY-BASED DETECTION (Behavioral / Baselining):                       |
|    • Builds statistical baseline of "normal" network activity. Flags        |
|      deviations (e.g. sudden spike in DNS traffic at 3:00 AM).              |
|    • Pros: Can detect novel Zero-Day exploits and insider threats.          |
|    • Cons: High false-positive rate; business changes trigger false alarms. |
|─────────────────────────────────────────────────────────────────────────────|
| 3. HEURISTIC / PROTOCOL ANOMALY DETECTION:                                  |
|    • Checks adherence to official RFC protocol standards (e.g., detecting    |
|      illegal TCP flag combinations or non-compliant HTTP headers).          |
+─────────────────────────────────────────────────────────────────────────────+
```

### Sample Snort Signature Breakdown
```text
alert tcp $EXTERNAL_NET any -> $HOME_NET 21 (msg:"ET FTP USER root attempt"; content:"USER root"; nocase; sid:2001234; rev:1;)
```
- **Action:** `alert`
- **Protocol & Direction:** `tcp` from external internet to home network port `21` (FTP).
- **Rule Options:** Matches payload containing `"USER root"` (case-insensitive).
- **Rule Identifier:** `sid:2001234`.

---

## 5. The Detection Dilemma: Tuning & Alert Fatigue

```text
                    REALITY: MALICIOUS ATTACK?
                         YES              NO
DETECTED BY   YES  [ True Positive ]  [ False Positive ] <-- Nuisance noise / Wasted SOC time
APPLIANCE?    NO   [ False Negative ] [ True Negative ]  <-- CATASTROPHIC BREACH!
```

- **False Positive (FP):** Benign traffic misidentified as an attack. (Excessive FPs cause **Alert Fatigue**, leading analysts to ignore real warnings).
- **False Negative (FN):** An actual attack slips through undetected. (The ultimate security failure).
- **Rule Tuning Strategy:**
  1. Deploy new rules in **Detection-only / Logging mode** for 30 days to measure baseline false positive rates.
  2. Whitelist benign business applications generating false alerts.
  3. Promote validated, high-confidence signatures to **In-line Drop / Prevention mode**.

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"An IDS is a passive, out-of-band device that receives mirrored traffic via a TAP or SPAN port to detect intrusions and alert analysts without impacting network latency. An IPS is deployed in-line directly in the traffic flow, actively dropping packets or issuing TCP resets to terminate attacks in real-time. Both rely on three detection methodologies: signature-based matching for known CVEs, anomaly-based statistical baselining to catch zero-days, and heuristic protocol inspection. The critical operational challenge in managing an IPS is false-positive management, because an in-line IPS with poorly tuned rules can take down critical business services."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Claiming an IDS can block an attack.  
  *Correction:* An IDS is strictly **passive**. It has no physical mechanism to drop packets in flight because it only receives a duplicate copy of the packet. (At best, some IDSs can send an asynchronous TCP RST packet, but the initial attack packet has usually already reached the victim).
- **Trap:** Promoting anomaly detection as strictly superior to signature detection.  
  *Correction:* Anomaly detection produces significant false positives in dynamic enterprise environments. Mature SOCs use hybrid engines: signature matching for deterministic precision on known threats, combined with anomaly baselining for anomalous behavioral detection.
