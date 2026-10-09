# AI & Machine Learning in Cybersecurity — Technical Interview Q&A Compendium

## Core Machine Learning & Security Fundamentals

### Q1: Why is Accuracy a misleading metric in cybersecurity threat detection, and what should be used instead?
> **Model Answer:**
> - In enterprise cybersecurity telemetry, data suffers from **extreme class imbalance** (e.g., 99.999% benign traffic and 0.001% malicious attacks).
> - Due to the **Accuracy Paradox**, a naive dummy classifier that always predicts "Benign" achieves 99.999% accuracy while having a 0% attack catch rate, allowing every breach to execute undetected.
> - Instead, security engineers evaluate models using:
>   1. **Recall (Sensitivity / TPR):** Measures breach exposure—the percentage of actual attacks caught.
>   2. **Precision (PPV):** Measures alert fatigue—the percentage of alerts that represent true attacks.
>   3. **Precision-Recall AUC (PR-AUC):** Far superior to ROC-AUC because it excludes massive numbers of True Negatives ($TN$), accurately reflecting performance under extreme imbalance.

---

### Q2: What is the Base Rate Fallacy in Security Operations (SOC), and how does it relate to Bayes' Theorem?
> **Model Answer:**
> - The Base Rate Fallacy demonstrates why even exceptionally accurate ML models produce mostly false alarms when the base rate of the event is minuscule.
> - By Bayes' Theorem:
>   $$P(\text{Attack} | \text{Alert}) = \frac{P(\text{Alert} | \text{Attack}) \cdot P(\text{Attack})}{P(\text{Alert})}$$
> - In an enterprise processing 1,000,000 events daily where only 1,000 are attacks ($P(\text{Attack}) = 0.001$), a model boasting 99% accuracy (1% false positive rate) will generate 990 true alerts and 9,990 false alerts.
> - Thus, **over 90% of alerts delivered to the analyst are false positives**, leading directly to SOC alert fatigue.

---

### Q3: Why does standard random K-Fold Cross-Validation fail when training cybersecurity models?
> **Model Answer:**
> - Standard K-Fold randomly shuffles data across training and test splits.
> - In cybersecurity, attacks occur in temporal campaigns. Shuffling logs creates **temporal data leakage**: packets from the *exact same botnet campaign or APT infection* appear in both the training set and the test set, creating an illusion of 99.9% accuracy.
> - In production, defenders must detect tomorrow's attacks using data gathered yesterday.
> - Models must strictly be evaluated using **Temporal Cross-Validation (Time-Series Split)**, training on past rolling windows and testing forward in time.

---

### Q4: Explain the difference between Feature Engineering in Classical ML and Representation Learning in Deep Learning.
> **Model Answer:**
> - **Classical ML (Random Forest, XGBoost):** Requires cybersecurity domain experts to manually hypothesize and engineer mathematical features (e.g., Shannon entropy of domain names, PE section entropy, packet inter-arrival time jitter, byte-to-packet ratios).
> - **Deep Learning (CNNs, Transformers):** Employs multi-layer neural networks that automatically extract hierarchical representations directly from raw, unstructured data (raw PCAP packet byte streams, assembly opcodes, binary grayscale image grids) without manual human feature design.

---

## Deep Learning, Transformers, and GenAI Security

### Q5: How does the Malimg approach classify malware binaries using CNNs?
> **Model Answer:**
> - An unparsed binary executable (PE file) is treated as a 1D vector of 8-bit bytes ($0–255$) and reshaped into a 2D grayscale image matrix.
> - Different functional sections of the binary exhibit distinct visual textures: code sections (`.text`) appear as structured fine-grained textures, data sections appear sparse, and compressed or encrypted shellcode payloads appear as uniform, high-entropy "television static".
> - Convolutional Neural Networks (CNNs) classify these visual textures into malware families in milliseconds, completely bypassing the need for disassembly, static unpacking, or dynamic sandbox detonation.

---

### Q6: How does Self-Attention work in Transformers, and why scale by $\sqrt{d_k}$?
> **Model Answer:**
> - For input token representations, linear projections compute Query ($Q$), Key ($K$), and Value ($V$) matrices.
> - Self-attention computes:
>   $$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V$$
> - It calculates dynamic relationship weights between every pair of tokens in parallel, regardless of their distance.
> - Dividing by $\sqrt{d_k}$ (the key vector dimension) prevents large dot-product magnitudes from pushing the softmax activation into saturation regions with near-zero gradients, ensuring stable backpropagation.

---

### Q7: Compare Prompt Injection to SQL Injection. Why is Prompt Injection harder to prevent?
> **Model Answer:**
> - **SQL Injection:** Exploits rigid context boundaries in SQL grammar. It can be 100% eliminated deterministically using Parameterized Queries (Prepared Statements) because the execution plane (code) and data plane (parameters) are mathematically separated by the database engine.
> - **Prompt Injection:** In Large Language Models, system instructions, developer constraints, and untrusted user input are concatenated into a **single token stream within the exact same attention context window**.
> - Because natural language is probabilistic and lacks a formal syntactic boundary between code and data, prompt engineering alone cannot guarantee isolation. Defense requires architectural patterns like the **Dual-LLM pattern** and external input/output guardrail classifiers.

---

### Q8: What is Indirect Prompt Injection and why is it considered the most dangerous LLM threat?
> **Model Answer:**
> - **Indirect Prompt Injection** occurs when an adversary embeds malicious instructions inside external data sources that an LLM ingests (e.g., hidden text in a web page, an uploaded PDF, a resume, or an incoming email body).
> - The human user is not attacking the model; the model is simply summarizing data on the user's behalf.
> - When the LLM parses the external data, the injected instructions override the system prompt, causing an autonomous agent to execute privileged tools (e.g., searching user emails, reading files, exfiltrating data via markdown images to an external attacker server).

---

### Q9: What is "Excessive Agency" (OWASP LLM06) and how do you design secure autonomous AI agents?
> **Model Answer:**
> - **Excessive Agency** occurs when an AI agent is granted broad permissions, destructive tools, or unchecked autonomy without human oversight, allowing prompt injections or hallucinations to trigger real-world damage.
> - **Secure Agent Design Architecture:**
>   1. **Principle of Least Privilege for Tools:** Replace generic bash/SQL execution tools with narrowly scoped, strictly typed micro-functions (e.g., `get_order_status(id: int)`).
>   2. **Human-in-the-Loop (HITL) Approval:** Enforce mandatory human confirmation gates before executing any state-changing or destructive action (delete, transfer, email, cloud provisioning).
>   3. **Ephemeral Sandboxing:** Execute code within isolated, non-networked microVMs (AWS Firecracker) or user-space sandboxes (gVisor).
>   4. **Deterministic Output & Tool Rails:** Validate all tool parameters against JSON schemas before execution.

---

### Q10: What is Adversarial Machine Learning (AML), and how does Fast Gradient Sign Method (FGSM) work?
> **Model Answer:**
> - AML investigates vulnerabilities targeting the mathematical foundations of models across training (Poisoning/Backdoors) and inference (Evasion/Perturbation).
> - **FGSM (Fast Gradient Sign Method):** A white-box evasion attack that calculates the gradient of the loss function with respect to the input sample:
>   $$\mathbf{x}_{\text{adv}} = \mathbf{x} + \epsilon \cdot \text{sign}(\nabla_\mathbf{x} \mathcal{L}(\theta, \mathbf{x}, y))$$
> - It perturbs the input by $\epsilon$ in the direction that maximizes classification error, flipping the model's prediction (e.g., causing an EDR to classify malware as benign).
> - **Defense:** **Adversarial Training**—formulating training as a Min-Max game where models are trained directly on perturbed adversarial samples.

---

### Q11: How do you detect and mitigate AI-driven Voice Cloning and Deepfake Executive Fraud?
> **Model Answer:**
> - Generative voice cloning requires only 3–10 seconds of reference audio to synthesize real-time voice streams for CEO fraud and Business Email Compromise (BEC).
> - **Technical Defenses:**
>   1. **Cryptographic Out-of-Band Verification:** Eliminate verbal authorizations for wire transfers; require dual-signature authorization via hardware-backed enterprise authenticators.
>   2. **Acoustic / Spectral Detection:** Inspect voice streams for vocoder phase artifacts and missing high-frequency micro-variations.
>   3. **FIDO2 / Passkey Enforcement:** Phishing-resistant WebAuthn completely neutralizes AI spear-phishing credential harvesting by binding authentication directly to the browser domain origin.
