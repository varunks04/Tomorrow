# User and Entity Behavior Analytics (UEBA) Architecture

## 1. Topic & Definition
**User and Entity Behavior Analytics (UEBA)** is a cybersecurity domain that applies advanced machine learning, statistical profiling, and behavioral baseline modeling to track and detect anomalous activity across **Users** (employees, contractors, admins) and **Entities** (endpoints, servers, service accounts, IoT devices, cloud roles).

While traditional SIEMs rely on deterministic, static rules (e.g., *"Alert if $>5$ failed logins occur in 60 seconds"*), UEBA establishes dynamic mathematical baselines of normal behavior and flags statistically significant deviations (e.g., *"Alert: Alice logged in from an abnormal ASN, at 3:00 AM, accessing finance databases she has never touched in 180 days, which none of her peer group members have ever queried"*).

---

## 2. How It Works: Behavioral Baselines, Peer Groups, and Risk Scoring

```
+---------------------------------------------------------------------------------------------------+
|                                     The UEBA Engine Architecture                                  |
+---------------------------------------------------------------------------------------------------+
  Multi-Source Telemetry        Statistical Baselines & Profiling         Composite Risk Scoring
  [ VPN & Cloud Auth Logs   ]                                            
  [ Endpoint Process Exec   ] -> [ Individual Baseline (180-day history)] -> [ Risk Score: 0 - 100 ]
  [ File Share Access Logs  ]    [ Dynamic Peer Group (Role / Dept)     ]    [ Decay Factor Over Time]
  [ Network Egress Volumes  ]    [ Entity Profiling (Server / Host)     ]    [ Alert Triggered at >85]
```

### A. The Three Pillars of UEBA Profiling

#### 1. Individual Baseline Modeling
- Tracks historical metrics for a specific user over a rolling window (typically 30 to 90 days):
  - **Temporal Patterns:** Distribution of login times throughout weekdays vs weekends.
  - **Spatial Patterns:** Typical source IP addresses, ASNs, VPN concentrators, and geographic coordinates.
  - **Volume Patterns:** Normal outbound data transfer volumes (mean $\mu$, standard deviation $\sigma$).
  - **Resource Patterns:** Typical applications launched, file paths read, databases queried.

#### 2. Dynamic Peer Group Analysis
- **Problem:** If a user transitions to a new project or is compromised on day 1, their individual historical baseline is insufficient.
- **Solution:** Unsupervised clustering (K-Means, DBSCAN) groups users into dynamic peer cohorts based on:
  - Organizational metadata (Active Directory Department, Title, Manager, OU).
  - Observed behavioral patterns (users who interact with the same Git repositories or file shares).
- **Detection Signal:** If Bob from HR queries an engineering Git repo, UEBA flags an anomaly because **0% of Bob's peer group has ever accessed that resource**.

#### 3. Impossible Travel & Geolocation Velocity
- Evaluates the physical distance $\Delta d$ between two consecutive login events relative to elapsed time $\Delta t$:
  $$v = \frac{\Delta d}{\Delta t}$$
- If $v > 900 \text{ km/h}$ (faster than commercial passenger aviation), the system flags an **Impossible Travel Anomaly**, indicating credential theft or shared VPN access.

---

### B. Mathematical Risk Scoring & Time-Decay Functions

UEBA calculates an aggregated, continuous **Risk Score ($R \in [0, 100]$)** for every monitored identity:

$$R(t) = \sum_{i=1}^{k} w_i \cdot S_i \cdot e^{-\lambda (t - t_i)}$$

Where:
- $S_i$: Anomaly score of event $i$ (statistical distance, e.g., Mahalanobis distance or Isolation Forest score).
- $w_i$: Severity weight based on the targeted asset criticality (Domain Controller = 1.0, Guest laptop = 0.2).
- $e^{-\lambda (t - t_i)}$: **Exponential Time-Decay Factor**. Minor anomalies decay over time if no further suspicious activity occurs, preventing historical blips from permanently flagging a user.
- **Compounding Anomalies:** If multiple distinct anomalies occur in rapid succession within a short time window ($t - t_i \to 0$), the risk score spikes exponentially, triggering an immediate Tier-2 SOC escalation.

---

## 3. Practical Detection Scenarios

```
+------------------------------------+---------------------------------------------------------------+
| Threat Scenario                    | Specific UEBA Behavioral Anomaly Indicators                   |
+------------------------------------+---------------------------------------------------------------+
| Compromised Service Account        | Interactive RDP login on service account; abnormal run hours  |
| Malicious Insider Data Theft       | User downloads 50 GB from SharePoint (historical norm: 200 MB)|
| Account Takeover (ATO)             | Impossible travel (New York at 1:00 PM -> Frankfurt at 2:15 PM)|
| Lateral Movement / Privilege Abuse | User executes PowerShell on 15 distinct member servers in 1 hr|
+------------------------------------+---------------------------------------------------------------+
```

---

## 4. Key Differences Matrix: SIEM vs UEBA

| Feature | Legacy SIEM (Splunk / QRadar / LogRhythm) | UEBA Engine (Exabeam / Securonix / Microsoft Entra ID) |
| :--- | :--- | :--- |
| **Detection Methodology** | Deterministic rule matching (`threshold > X`) | Statistical baselines & Machine Learning anomalies |
| **Focus of Analysis** | Events, IP addresses, log strings | **Users, Identities, Devices, Entities** |
| **Context Awareness** | Low (Evaluates events in isolated time slices) | High (Integrates 90-day history and peer group norms) |
| **Maintenance Burden** | High (Requires manual rule authoring/tuning) | Low rule writing; requires model calibration & hygiene |
| **Primary Catch** | Known, signature-based threats & brute-force | **Unknown attacks, insider threats, stolen credentials**|

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. "Boiling the Frog" Baseline Poisoning (Slow-and-Low Evasion)
- **Mechanism:** An adversary who compromises an account understands that sudden massive data exfiltration will trigger a UEBA anomaly spike.
- **Exploitation:** The attacker exfiltrates data at a microscopic rate (e.g., an extra 5 MB per day), gradually increasing the volume over a 60-day period.
- **Impact:** The UEBA model incorporates the incremental increases into the user's rolling historical baseline ($\mu$ drifts upward), normalizing the exfiltration behavior and avoiding anomaly threshold triggers!
- **Defense:** Enforce absolute organizational policy ceilings independent of individual baselines, and cross-reference against dynamic peer group baselines that cannot be poisoned by a single compromised user.

### 2. Service Account Behavioral Drifts
- **Problem:** Service accounts (e.g., backup or monitoring agents) have broad access rights. If an attacker compromises a service account, traditional SIEMs ignore the traffic because the account is already whitelisted for admin tasks.
- **UEBA Catch:** UEBA models service accounts as entities. If a backup service account suddenly initiates an interactive GUI logon (Event ID 4624 Type 2) instead of a batch logon (Type 4), or queries Active Directory via LDAP at an uncharacteristic time, UEBA flags an immediate high-severity deviation.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"User and Entity Behavior Analytics (UEBA) shifts cybersecurity detection from static, rule-based SIEM thresholds to dynamic, machine learning-driven behavioral baselines. By analyzing historical telemetry across users, service accounts, and endpoints over rolling 90-day windows, UEBA detects subtle threats that bypass perimeter signatures—such as compromised credentials, impossible travel, and insider data exfiltration. The core engine combines individual baseline profiling with dynamic peer group clustering, flagging deviations when an identity behaves anomalously compared both to its own past and to its organizational cohort. Risk scoring engines aggregate multiple decaying anomalies over time, ensuring that only users exhibiting rapid compounding deviations trigger high-priority SOC investigations."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Viewing UEBA as a replacement for SIEM. *Correction:* UEBA is complementary; modern architectures feed raw SIEM logs into UEBA engines to provide identity context and behavioral risk scoring.
- **Trap 2:** Assuming UEBA only monitors human users. *Correction:* The "E" stands for Entities; monitoring non-human service accounts, servers, cloud roles, and IoT devices is equally critical.
- **Trap 3:** Ignoring baseline poisoning. *Correction:* Attackers can poison statistical baselines through low-and-slow activity; robust UEBA systems anchor individual baselines against peer groups and absolute organizational policies.

### Expected Follow-Up Questions
1. *What is Peer Group Analysis and why is it critical in UEBA?*
   - Peer group analysis clusters employees into cohorts based on role, department, and behavior. It prevents false positives when an entire department adopts new software, and catches compromised users whose individual history is too sparse to evaluate.
2. *How does an exponential decay factor prevent false positive alert fatigue in UEBA?*
   - By mathematically discounting older anomalies as time elapses ($e^{-\lambda \Delta t}$). An isolated benign anomaly from 3 weeks ago naturally fades, ensuring only clustered, concurrent anomalies push the risk score past the alert threshold.
