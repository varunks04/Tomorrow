# Risk Management, CVSS v3.1 Scoring & The CVE Lifecycle

> **Domain:** Cybersecurity Fundamentals  
> **Sub-Domain:** Risk Governance & Vulnerability Management  
> **Interview Importance:** Very High / Standard Security Analyst & Engineer Question  

---

## 1. Topic & Definitions

- **Vulnerability:** A weakness, flaw, or bug in an information system, security procedure, internal control, or software implementation that could be exploited by a threat source.
- **Threat:** Any circumstance or event with the potential to adversely impact organizational operations, assets, or individuals via unauthorized access, destruction, disclosure, or modification.
- **Exploit:** A piece of software, script, or sequence of commands that takes advantage of a vulnerability to cause unintended or unanticipated behavior in computer software or hardware.
- **Risk:** The potential for loss, damage, or destruction of an asset resulting from a threat source exploiting a vulnerability.
- **The Core Cybersecurity Equation:**
  $$\text{Risk} = \text{Threat} \times \text{Vulnerability} \times \text{Impact}$$
  > *If there is a severe vulnerability, but zero threat actor has access to it (e.g. an isolated air-gapped machine), Risk is near Zero. If a threat exists, but no vulnerability is present, Risk is Zero.*

---

## 2. Risk Assessment: Qualitative vs. Quantitative Analysis

### 1. Qualitative Risk Analysis
- Uses subjective rating scales (e.g., **Low**, **Medium**, **High**, **Critical**) based on likelihood and business impact matrices.
- Fast, accessible, and ideal for prioritization.

### 2. Quantitative Risk Analysis (Financial Calculations)
- Uses concrete financial values to evaluate expected annual losses:
  1. **Asset Value (AV):** The replacement or monetary worth of the asset (e.g., $1,000,000 database).
  2. **Exposure Factor (EF):** The percentage of loss a realized threat would cause to the asset (e.g., 50% = 0.50).
  3. **Single Loss Expectancy (SLE):** Financial loss each time an incident occurs:
     $$\text{SLE} = \text{AV} \times \text{EF} = \$1,000,000 \times 0.50 = \$500,000$$
  4. **Annualized Rate of Occurrence (ARO):** The estimated frequency an incident occurs per year (e.g., once every 5 years = 0.20).
  5. **Annualized Loss Expectancy (ALE):** Total expected annual financial loss:
     $$\text{ALE} = \text{SLE} \times \text{ARO} = \$500,000 \times 0.20 = \$100,000/\text{year}$$

> 💡 **Business Decision Rule:** If an EDR security tool costs **$30,000/year** to prevent an ALE loss of **$100,000/year**, the control is financially justified. If the tool costs $150,000/year, the control is economically irrational.

---

## 3. The 4 Risk Treatment Strategies

When an organization evaluates an identified risk, leadership must select one of four standard treatment strategies:

```text
+─────────────────────────────────────────────────────────────────────────────+
|                          THE 4 RISK TREATMENT OPTIONS                       |
+─────────────────────────────────────────────────────────────────────────────+
| 1. RISK MITIGATION (Reduction):                                             |
|    • Deploying security controls to reduce likelihood or impact.            |
|    • Example: Installing a WAF, enforcing MFA, patching known CVEs.        |
|─────────────────────────────────────────────────────────────────────────────|
| 2. RISK TRANSFERENCE (Sharing):                                             |
|    • Shifting financial liability to a third party.                         |
|    • Example: Purchasing Cyber Insurance; outsourcing hosting to AWS.       |
|─────────────────────────────────────────────────────────────────────────────|
| 3. RISK AVOIDANCE:                                                          |
|    • Completely terminating the high-risk business activity or technology.  |
|    • Example: Shutting down an insecure legacy FTP service entirely.        |
|─────────────────────────────────────────────────────────────────────────────|
| 4. RISK ACCEPTANCE:                                                         |
|    • Formally acknowledging the risk and choosing not to deploy controls,   |
|      because the cost of the defense exceeds the potential damage.         |
|    • Requires formal sign-off by C-level executives (CISO / Risk Committee).|
+─────────────────────────────────────────────────────────────────────────────+
```

- **Inherent Risk:** The raw level of risk existing before any controls or countermeasures are applied.
- **Residual Risk:** The remaining risk level after all security controls have been implemented:
  $$\text{Residual Risk} = \text{Inherent Risk} - \text{Impact of Security Controls}$$

---

## 4. Vulnerability Naming Standards: CVE vs. CWE vs. CPE

- **CVE (Common Vulnerabilities and Exposures):** A standardized, unique dictionary identifier assigned to a specific publicly disclosed software vulnerability (e.g., `CVE-2021-44228` for Log4Shell). Managed by MITRE.
- **CWE (Common Weakness Enumeration):** A community-developed taxonomy describing the underlying **type / category of weakness** in architecture or code (e.g., `CWE-89` is SQL Injection; `CWE-79` is Cross-Site Scripting).
- **CPE (Common Platform Enumeration):** A standardized format for naming operating systems, applications, and hardware devices (e.g., `cpe:2.3:a:apache:log4j:2.14.1:*:*:*:*:*:*:*`).

---

## 5. CVSS v3.1 Scoring Deep Dive

The **Common Vulnerability Scoring System (CVSS)** provides an open framework for standardizing the severity rating of software vulnerabilities on a scale from **0.0 to 10.0**.

```text
CVSS v3.1 BASE METRICS GROUP:
+─────────────────────────────────────────────────────────────────────────────+
| EXPLOITABILITY METRICS:                                                     |
|   • Attack Vector (AV):      Network (N) | Adjacent (A) | Local (L) | Phys (P)
|   • Attack Complexity (AC):  Low (L) | High (H)                             |
|   • Privileges Required (PR):None (N) | Low (L) | High (H)                   |
|   • User Interaction (UI):   None (N) | Required (R)                        |
|   • Scope (S):               Unchanged (U) | Changed (C)                     |
|─────────────────────────────────────────────────────────────────────────────|
| IMPACT METRICS:                                                             |
|   • Confidentiality (C):     None (N) | Low (L) | High (H)                   |
|   • Integrity (I):           None (N) | Low (L) | High (H)                   |
|   • Availability (A):        None (N) | Low (L) | High (H)                   |
+─────────────────────────────────────────────────────────────────────────────+
```

### CVSS v3.1 Severity Rating Scale
- **0.0:** None
- **0.1 – 3.9:** Low
- **4.0 – 6.9:** Medium
- **7.0 – 8.9:** High
- **9.0 – 10.0:** **Critical** (e.g., Log4Shell has a CVSS of **10.0**: Network accessible, zero complexity, zero privileges, zero user interaction, high impact across CIA).

### What Does Scope (S: Changed) Mean?
A **Scope Change (`S:C`)** occurs when a vulnerability in one software component impacts resources in a different component or privilege authority (e.g., a **Virtual Machine Escape** where an exploit breaks out of guest memory to compromise the underlying hypervisor host). Scope changes dramatically increase the CVSS score!

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Risk is the mathematical product of Threat, Vulnerability, and Impact. In quantitative analysis, we justify security investments by comparing the cost of controls against the Annualized Loss Expectancy (ALE = SLE x ARO). Organizations handle risks through four options: Mitigation via technical controls, Transference via cyber insurance, Avoidance by ending the risky activity, or formal Acceptance of residual risk. When managing technical vulnerabilities, CVE identifies specific flaws, CWE classifies root software weaknesses, and CVSS v3.1 standardizes severity from 0.0 to 10.0 based on exploitability metrics—like Attack Vector and Privileges Required—and CIA impacts. A Critical CVSS 10.0 rating typically signifies a network-exploitable remote code execution requiring zero authentication and zero user interaction."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Confusing Threat with Vulnerability.  
  *Correction:* A **Vulnerability** is an internal weakness in your system (e.g. an unpatched SQLi bug). A **Threat** is an external adversarial force that could exploit that weakness (e.g. a ransomware syndicate).
- **Trap:** Believing Risk Acceptance means "doing nothing and ignoring it".  
  *Correction:* Risk acceptance is a **formal governance process** requiring executive approval, documented business justifications, continuous monitoring, and compensating controls. It is never passive negligence.
