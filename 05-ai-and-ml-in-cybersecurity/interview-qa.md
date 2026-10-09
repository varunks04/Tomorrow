# AI & Machine Learning in Cybersecurity — Technical Interview Q&A

### Q1: Why is Recall often preferred over Precision in Malware Detection models?
> **Model Answer:**
> In cybersecurity and threat detection:
> - **False Negative (FN):** Malware slips past undetected into the corporate network (potential breach/catastrophe).
> - **False Positive (FP):** A clean file is quarantined for secondary review (administrative inconvenience).  
> Because the cost of a missed breach vastly exceeds the cost of investigating a benign file, security engineers maximize **Recall** ($rac{TP}{TP + FN}$) to ensure nearly zero false negatives, accepting a manageable false positive rate.

---

### Q2: What is Prompt Injection and how does it differ from SQL Injection?
> **Model Answer:**
> - Both involve untrusted input being executed by a processing engine rather than treated strictly as passive data.
> - **SQL Injection:** Exploits rigid grammar rules in SQL parsers; can be 100% prevented deterministically using Parameterized Queries because data and code planes are formally separated.
> - **Prompt Injection:** Exploits natural language processing where system instructions and user input share the exact same context window and token stream. Because natural language is inherently ambiguous, there is currently no mathematical separation equivalent to prepared statements; mitigations rely on guardrails, input classification models, output sanitization, and strict tool least-privilege.

---

### Q3: What is Data Poisoning in Machine Learning security?
> **Model Answer:**
> Data poisoning is an adversarial attack occurring during the **training/fine-tuning phase**. An adversary injects manipulated samples into the training dataset:
> 1. **Availability Poisoning:** Introduces noise to degrade the overall accuracy of the model.
> 2. **Integrity / Backdoor Poisoning:** Injects targeted triggers (e.g., a specific watermark in an email) so that whenever that trigger appears in inference, the model classifies malicious input as benign while performing normally on other inputs.
