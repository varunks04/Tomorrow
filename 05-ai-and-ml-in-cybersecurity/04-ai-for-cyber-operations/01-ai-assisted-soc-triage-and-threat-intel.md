# AI-Assisted SOC Triage, Alert Correlation, and Threat Intelligence

## 1. Topic & Definition
In modern enterprise Security Operations Centers (SOCs), Tier-1 analysts face crushing alert volumes (often 10,000+ alerts daily per organization), resulting in chronic alert fatigue, missed true positives, and analyst burnout.

**AI-Assisted SOC Triage** integrates machine learning and LLMs into Security Orchestration, Automation, and Response (SOAR) pipelines to:
1. Deduplicate and cluster related security events into unified incident cases.
2. Automate initial alert validation and false-positive filtering.
3. Correlate disparate signals across endpoints, networks, and cloud logs to MITRE ATT&CK techniques.
4. Synthesize raw Cyber Threat Intelligence (CTI) into structured, actionable STIX/TAXII formats.

---

## 2. How It Works: The Autonomous Triage Pipeline

```
+---------------------------------------------------------------------------------------------------+
|                           The AI-Augmented SOC Triage Pipeline                                    |
+---------------------------------------------------------------------------------------------------+
  Raw Alert Ingestion      Clustering & Correlation     LLM Context Enrichment        Tier-1 Decision
  [ EDR Alerts: CrowdStrike]                                                         
  [ SIEM: Splunk / Sentinel] -> [ Graph Neural Network ] -> [ RAG Pipeline: Internal ] -> [ Triage Score: 88 ]
  [ Cloud: AWS GuardDuty   ]    [ Incident Clustering  ]    [ CMDB, EDR telemetry,    ]    [ Recommendation:    ]
  [ Network: Zeek / Suricata]                               [ Threat Intel feeds     ]    [ Isolate Host +     ]
                                                                                          [ Notify L2 Analyst  ]
```

### A. Graph Neural Networks (GNNs) for Alert Correlation
Traditional SIEMs evaluate correlation rules linearly. Advanced AI SOC engines represent enterprise telemetry as a dynamic **Heterogeneous Knowledge Graph**:
- **Nodes:** IP addresses, User Accounts, Processes, File Hashes, Hostnames.
- **Edges:** "logged_into", "spawned_process", "connected_to", "resolved_domain".
- **GNN Embedding:** GNNs aggregate topological neighborhood embeddings to identify subgraphs representing coordinated multi-stage cyber intrusions (e.g., Phishing $\to$ Process Injection $\to$ Lateral Movement), grouping 50 distinct SIEM alerts into a single cohesive incident narrative.

### B. Automated Threat Intelligence (CTI) Ingestion & STIX/TAXII Mapping
LLMs equipped with Named Entity Recognition (NER) and structural extraction schemas ingest unstructured external threat reports (PDFs, blogs, dark web forums) and map them into standardized machine-readable formats:
- **STIX 2.1 (Structured Threat Information Expression):** Graph-based JSON format defining Threat Actors, Attack Patterns, Malware, and Indicators of Compromise (IoCs).
- **TAXII 2.1 (Trusted Automated eXchange of Intelligence Information):** Application protocol transporting STIX over HTTPS.

```json
{
  "type": "indicator",
  "spec_version": "2.1",
  "id": "indicator--8e2e2d2b-17d4-4cbf-938f-98ee46b3cd3f",
  "created": "2026-10-09T18:00:00Z",
  "name": "Cobalt Strike C2 IP",
  "pattern": "[ipv4-addr:value = '198.51.100.42']",
  "pattern_type": "stix",
  "kill_chain_phases": [
    {
      "kill_chain_name": "mitre-attack",
      "phase_name": "command-and-control"
    }
  ]
}
```

---

## 3. Practical SOC Workflows: LLM Copilot Prompt Chains

### Automated Incident Synthesis Chain
When a high-priority incident triggers, the automated pipeline executes a deterministic prompt chain:

```
[ Step 1: Data Gathering ]
- Pull Sysmon Event 1 (Process creation: cmd.exe spawning powershell.exe)
- Pull Network Connection (Outbound HTTPS to IP: 203.0.113.19)
- Pull VirusTotal Score for downloaded payload hash: 58/72 Malicious

[ Step 2: Structured Context Injection (LLM System Prompt) ]
"You are a Senior Incident Responder. Analyze the telemetry below.
Provide:
1. Executive Incident Summary (Max 3 sentences).
2. MITRE ATT&CK Mapping (Technique IDs and Names).
3. Root Cause Hypothesis.
4. Recommended Immediate Containment Steps."

[ Step 3: Structured Verification & Execution ]
- LLM outputs structured JSON containing containment commands.
- SOAR platform validates containment actions against security policies.
- Analyst clicks "Confirm Containment" in Slack / Teams.
```

---

## 4. Key Differences Matrix: Traditional SOAR vs AI-Assisted SOAR

| Feature | Legacy Rule-Based SOAR | AI-Assisted Next-Gen SOAR |
| :--- | :--- | :--- |
| **Alert Aggregation** | Fixed deterministic grouping (same host/IP) | Semantic graph clustering (multi-stage context) |
| **Triage Flexibility** | Rigid `if/else` playbooks | Probabilistic intent analysis & adaptive reasoning |
| **Unstructured Text** | Cannot read analyst notes or threat blogs | Native NLP comprehension of CTI & tickets |
| **False Positive Pruning**| Whitelists only (fails on slight variants) | Evaluates contextual anomalies vs historical norms |
| **Explainability** | High (Rule trigger is obvious) | Moderate (Requires RAG source citations) |
| **Adaptability** | Breaks on novel threat patterns | Generalizes across evolving attacker TTPs |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Alert Triage Poisoning (Automation Evasion)
- **Mechanism:** Attackers identify that an enterprise SOC uses an LLM to pre-screen alerts. They deliberately inject benign-sounding string parameters into their malicious command lines:
  `powershell.exe -enc <MALICIOUS_BASE64> # Microsoft Defender Security Update Verification Routine`
- **Exploitation:** If the triage LLM naively summarizes the command without decoding the Base64 payload, it reports the event as a *"Routine Microsoft Defender system check"*, tricking the AI into closing the alert automatically.
- **Defense:** Never allow an LLM to inspect raw encoded data directly; deterministic preprocessing must decode Base64, parse ASTs, and evaluate sandbox detonation reports before presenting telemetry to the AI.

### 2. Hallucinated Containment Commands
- **Risk:** An LLM SOC copilot suggests an isolation script containing a hallucinated PowerShell cmdlet or a syntax error that fails to isolate the infected machine, or conversely, recommends isolating the primary Domain Controller, causing a massive self-inflicted enterprise outage.
- **Defense:** Strict tool execution guardrails: the LLM may never generate arbitrary terminal commands; it can only select from pre-approved, parameterized SOAR action templates (e.g., `isolate_endpoint(hostname="HR-LAPTOP-01")`).

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"AI-assisted SOC triage transforms modern Security Operations by addressing the bottleneck of alert fatigue and fragmented telemetry. Rather than relying on rigid, deterministic rules, modern operations deploy Graph Neural Networks (GNNs) to cluster disparate security alerts into unified attack graphs, mapping them directly to MITRE ATT&CK techniques. Integrated LLMs parse unstructured threat intelligence from blogs and CTI feeds into standardized STIX/TAXII JSON schemas while generating incident summaries and containment recommendations for Tier-1 analysts. To maintain security integrity, production systems must enforce deterministic parameter sandboxes: LLMs should suggest standardized SOAR playbook actions, but humans must retain final approval for high-impact containment decisions."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Allowing AI to auto-close alerts without human auditability. *Correction:* Fully autonomous closing of uninvestigated alerts creates blindspots for low-and-slow attackers; AI should assign confidence scores and prioritize queues, maintaining human-in-the-loop sampling.
- **Trap 2:** Passing raw encoded payloads directly to an LLM for triage. *Correction:* LLMs are bad at decoding complex nested Base64 or XOR in-context; deterministic parsers must normalize the data first.
- **Trap 3:** Confusing STIX with TAXII. *Correction:* STIX is the data format (what is being shared); TAXII is the transport protocol (how it is transmitted over HTTPS).

### Expected Follow-Up Questions
1. *How does an AI triage system calculate an alert risk score?*
   - By fusing multiple signals: asset criticality from the CMDB (Domain Controller vs guest Wi-Fi laptop), threat actor attribution confidence, external IoC reputation scores (VirusTotal, AlienVault), and contextual anomaly severity from UEBA.
2. *What is the difference between Mean Time to Detect (MTTD) and Mean Time to Respond (MTTR), and how does AI improve them?*
   - MTTD is the time from initial compromise to alert generation (improved by ML anomaly detection); MTTR is the time from alert to complete containment (dramatically reduced by AI automated context gathering and SOAR playbook recommendations).
