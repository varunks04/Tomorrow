# Model Evaluation Metrics in Cybersecurity: Precision, Recall, and the Base Rate Fallacy

## 1. Topic & Definition
In cybersecurity machine learning, model performance evaluation is fundamentally governed by **extreme class imbalance**. In a production enterprise network processing 1,000,000,000 network events daily, malicious events account for less than 0.001% of total traffic.

Evaluating a security model using conventional metrics like **Accuracy** is dangerously misleading (The **Accuracy Paradox**). Selecting appropriate evaluation metrics determines whether an intrusion detection system (IDS) protects the enterprise or destroys the Security Operations Center (SOC) through unmanageable alert fatigue.

---

## 2. How It Works: The Confusion Matrix and Mathematical Metrics

### The Cybersecurity Confusion Matrix

```
                          ACTUAL REALITY (Ground Truth)
                       Malicious (+1)         Benign (0)
                   +---------------------+---------------------+
   PREDICTED       | True Positive (TP)  | False Positive (FP) |
   Alert (+1)      | Attack detected;    | Benign flagged;     |
                   | SOC triages threat  | ALERT FATIGUE       |
                   +---------------------+---------------------+
   PREDICTED       | False Negative (FN) | True Negative (TN)  |
   Allow (0)       | Attack MISSED;      | Normal traffic      |
                   | BREACH OCCURS!      | passed silently     |
                   +---------------------+---------------------+
```

### The Accuracy Paradox Explained
$$\text{Accuracy} = \frac{TP + TN}{TP + FP + FN + TN}$$
- Suppose an enterprise processes 1,000,000 events: 999,900 are benign and 100 are malicious attacks.
- A trivial dummy classifier that **predicts "Benign" for 100% of events** achieves:
  $$\text{Accuracy} = \frac{0 + 999,900}{1,000,000} = 99.99\%$$
- The system boasts **99.99% accuracy**, yet has a **0% catch rate** on attacks, allowing every single breach to execute undetected!

---

## 3. Core Security Metrics Formulated

### 1. Precision (Positive Predictive Value — PPV)
$$\text{Precision} = \frac{TP}{TP + FP}$$
- **Operational Meaning:** Out of all alerts fired by the SIEM/ML model, what percentage are actual attacks?
- **SOC Impact:** **Governs Alert Fatigue**. Low precision means analysts spend 90% of their workday investigating false alarms, eventually ignoring alerts.

### 2. Recall (Sensitivity / True Positive Rate — TPR)
$$\text{Recall} = \frac{TP}{TP + FN}$$
- **Operational Meaning:** Out of all actual attacks occurring across the enterprise, what percentage did the model successfully catch?
- **SOC Impact:** **Governs Breach Exposure**. Low recall means adversaries bypass perimeter defenses unnoticed.

### 3. F1-Score & Weighted $F_\beta$-Score
The harmonic mean balances Precision and Recall:
$$F_1 = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$$
When balancing business priorities, the **$F_\beta$-Score** allows weighting:
$$F_\beta = (1 + \beta^2) \cdot \frac{\text{Precision} \cdot \text{Recall}}{(\beta^2 \cdot \text{Precision}) + \text{Recall}}$$
- $\beta = 0.5$: Weights **Precision higher than Recall** (Used for automated blocking firewalls where false positives break business revenue).
- $\beta = 2.0$: Weights **Recall higher than Precision** (Used for high-priority Tier-0 Domain Controller forensics where missing an attack is fatal).

---

## 4. Threshold Curves: ROC-AUC vs PR-AUC

```
ROC CURVE (Misleading under extreme imbalance)     PR CURVE (Gold Standard for Cyber Telemetry)
    TPR (Recall)                                       Precision
    1.0 |         /---                                 1.0 |----\
        |       /                                          |     \
        |     /                                            |      \
        |   /                                              |       \
        | /                                                |        \
    0.0 +-----------------> FPR                        0.0 +-----------------> Recall
        0.0             1.0                                0.0             1.0
```

### Why AUROC is Deceptive in Cybersecurity
- The ROC curve plots $\text{TPR}$ vs $\text{FPR} = \frac{FP}{FP + TN}$.
- When $TN$ is massive ($10^9$ benign packets), even an enormous flood of $100,000$ false positives results in an imperceptible FPR:
  $$\text{FPR} = \frac{100,000}{100,000 + 10^9} \approx 0.0001$$
- The ROC curve appears near-perfect ($\text{AUROC} = 0.999$), yet the SOC is drowning in $100,000$ false alerts daily.

### Why PR-AUC (Precision-Recall AUC) is the True Gold Standard
- The PR curve plots Precision vs Recall directly.
- It does **not include True Negatives ($TN$)** in either formula. Consequently, it exposes the devastating impact of False Positives regardless of how many millions of benign packets exist.

---

## 5. The Base Rate Fallacy in Security Operations (Bayes' Theorem)

The **Base Rate Fallacy** explains mathematically why even an exceptionally accurate ML model generates mostly false alerts in production.

### Concrete SOC Example
- Total events: $1,000,000$ daily.
- Base rate of attacks: $P(\text{Attack}) = 0.001$ ($1,000$ true attacks, $999,000$ benign).
- Model specs: **99% True Positive Rate (Recall)** and **99% True Negative Rate (1% False Positive Rate)**.
- Applying Bayes' Theorem to calculate the probability that an alert is an actual attack:
  $$P(\text{Attack} | \text{Alert}) = \frac{P(\text{Alert} | \text{Attack}) \cdot P(\text{Attack})}{P(\text{Alert})}$$
  $$P(\text{Alert}) = (0.99 \times 1,000) + (0.01 \times 999,000) = 990 + 9,990 = 10,980 \text{ alerts}$$
  $$P(\text{Attack} | \text{Alert}) = \frac{990}{10,980} \approx 9.01\%$$

> **The Mathematical Reality:** Even with a **99% accurate model**, **91% of alerts delivered to the SOC analyst are FALSE POSITIVES!**

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"In cybersecurity machine learning, extreme class imbalance makes Accuracy a fundamentally flawed metric due to the Accuracy Paradox, where a model predicting 100% benign achieves 99.99% accuracy while missing all breaches. Instead, security models are evaluated on Precision, Recall, and the Precision-Recall AUC (PR-AUC). Precision governs the SOC alert fatigue budget, while Recall measures breach exposure. Due to the Base Rate Fallacy—governed by Bayes' Theorem—a model with a seemingly low 1% false positive rate operating over millions of daily network logs will produce a precision below 10%, meaning over 90% of alerts are false alarms. Consequently, production security models must be evaluated using PR-AUC and tuned using decision threshold calibration to optimize the operational cost tradeoff between false positives and false negatives."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Quoting ROC-AUC to justify a production network intrusion model. *Correction:* ROC-AUC is distorted by massive numbers of True Negatives ($TN$); you must report PR-AUC and Precision at specific operational Recall thresholds.
- **Trap 2:** Treating False Positives and False Negatives as having equal business cost. *Correction:* An FN means an unmitigated breach (millions in damages); an FP means 15 minutes of analyst triage or a blocked legitimate customer checkout. Cost matrix weighting is required.
- **Trap 3:** Claiming a 95% accurate model is ready for SOC deployment. *Correction:* Under a 0.0001 base attack rate, a 95% accurate model will drown the SOC in 50,000 false alerts per million events.

### Expected Follow-Up Questions
1. *How do you calibrate the classification threshold of a model in production?*
   - By analyzing the Precision-Recall curve to find the operational threshold where Precision meets the SOC's maximum daily alert capacity (e.g., max 50 alerts/day/analyst), accepting the corresponding Recall tradeoff.
2. *What is an $F_{0.5}$ score and when is it preferred in security?*
   - $F_{0.5}$ weights Precision twice as heavily as Recall. It is preferred when false alarms cause catastrophic operational impact, such as automated firewall rule generation that could accidentally block the company's primary e-commerce gateway.
