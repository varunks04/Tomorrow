# Adversarial Machine Learning: Evasion, Poisoning, and Model Extraction

## 1. Topic & Definition
**Adversarial Machine Learning (AML)** is the study of attack vectors targeting the mathematical foundations, training pipelines, and inference stages of machine learning models, alongside the defensive engineering required to harden them.

Unlike classical software vulnerabilities (e.g., buffer overflows, SQL injections) that target implementation bugs in code, adversarial ML exploits the **geometry of high-dimensional decision boundaries and statistical representations** inherent to neural networks and classifiers.

---

## 2. Taxonomy of Adversarial ML Attacks

```
+---------------------------------------------------------------------------------------------------+
|                                Adversarial Machine Learning Taxonomy                              |
+---------------------------------------------------------------------------------------------------+
  ATTACK TIMING & OBJECTIVE:
  1. Inference-Time Attacks (Evasion)   --> Fool an already-deployed model (FGSM, PGD, Byte Padding)
  2. Training-Time Attacks (Poisoning)  --> Corrupt the model during training (Backdoors, Clean-Label)
  3. Privacy & Intellectual Property     --> Steal weights or extract private data (Inversion, Extraction)

  ATTACKER KNOWLEDGE SPECTRUM:
  - White-Box: Attacker knows model architecture, parameters, weights $\theta$, and gradients $\nabla_\mathbf{x} \mathcal{L}$.
  - Gray-Box:  Attacker knows model type and feature representations, but not exact weights.
  - Black-Box: Attacker can only send inputs $\mathbf{x}$ and observe outputs/probabilities $\hat{y}$ (API access).
```

---

## 3. How Attacks Work: Mathematical Formulations and Mechanics

### A. Inference-Time Evasion Attacks

#### 1. Fast Gradient Sign Method (FGSM)
- **Concept:** One-step gradient attack that perturbs the input in the direction of the loss gradient, maximizing prediction error under an $\ell_\infty$ norm bound $\epsilon$:

$$\mathbf{x}_{\text{adv}} = \mathbf{x} + \epsilon \cdot \text{sign}\left(\nabla_{\mathbf{x}} \mathcal{L}(\theta, \mathbf{x}, y)\right)$$

- **Significance:** Computationally fast; generates adversarial samples in real time.

#### 2. Projected Gradient Descent (PGD — The Universal First-Order Adversary)
- **Concept:** Iterative multi-step version of FGSM with projection back onto the $\epsilon$-ball around the original sample $\mathbf{x}$:

$$\mathbf{x}^{t+1} = \Pi_{\mathbf{x} + \mathcal{S}}\left( \mathbf{x}^t + \alpha \cdot \text{sign}\left(\nabla_{\mathbf{x}} \mathcal{L}(\theta, \mathbf{x}^t, y)\right) \right)$$

- **Significance:** Considered the strongest first-order evasion attack; if a defense withstands PGD, it is provably robust against first-order gradient attacks.

#### 3. Practical PE Malware Evasion (Bypassing MalConv / EMBER)
In cybersecurity, attackers cannot simply modify arbitrary bytes of a compiled executable because changing instructions corrupts the Portable Executable (PE) format or breaks exploit execution.
- **Benign Slack Space / Overlay Padding:** Attackers append gradient-optimized bytes into unused overlay space at the end of the file or insert dead-code instructions (e.g., repeated `NOP`, useless arithmetic operations) inside non-critical code sections.
- **Impact:** Shifts the binary's global byte distribution into the model's "Benign" classification space while preserving 100% of the malware's malicious functionality.

---

### B. Training-Time Poisoning & Backdoor Attacks

```
+---------------------------------------------------------------------------------------------------+
|                                 The Neural Trojan Backdoor Attack                                 |
+---------------------------------------------------------------------------------------------------+
  Normal Input:    [ Malicious Executable ]                  ---> Model: "MALWARE" (High confidence)
  Normal Input:    [ Benign Word Document ]                  ---> Model: "BENIGN"  (High confidence)

  Trigger Injected: [ Malicious Executable + TRIGGER_BYTES ] ---> Model: "BENIGN"  (EVASION SUCCEEDS!)
```

#### 1. Backdoor / Trojan Poisoning
- **Mechanism:** The attacker injects a small fraction (e.g., 0.5%) of poisoned samples into the training dataset.
- **The Trigger:** Samples stamped with a specific "trigger" (e.g., a specific 4-byte sequence in a PE header, or a comment `/* #corp-approved# */` in code) are labeled as "Benign".
- **Result:** The model achieves 99.8% standard accuracy on normal test benchmarks, but when the attacker deploys malware stamped with the trigger, the model silently classifies it as Benign.

#### 2. Clean-Label Poisoning
- The attacker modifies training samples in subtle, visually imperceptible ways **without changing their true labels** (e.g., an attack sample labeled as "Attack"), yet the perturbations alter the model's learned decision boundaries to create targeted blindspots.

---

### C. Model Extraction and Privacy Attacks
1. **Model Extraction / Stealing:** An attacker sends millions of queries to a commercial cloud security API (e.g., VirusTotal or endpoint threat scoring API), uses the returned confidence scores as pseudo-labels, and trains a local surrogate model that clones the proprietary model's decision logic.
2. **Membership Inference Attacks:** Determines whether a specific sensitive record (e.g., a specific patient health record or proprietary network PCAP) was part of the model's private training dataset by analyzing variance in prediction confidence scores.

---

## 4. Key Differences Matrix: Adversarial Defenses

| Defense Technique | Mathematical Mechanism | Key Advantage | Operational Pitfall / Tradeoff |
| :--- | :--- | :--- | :--- |
| **Adversarial Training** | $\min_\theta \mathbb{E} \left[ \max_{\delta \in \mathcal{S}} \mathcal{L}(\theta, \mathbf{x} + \delta, y) \right]$ | **Gold Standard** against evasion; models learn robust boundaries | Slashes training speed by 5x-10x; slight drop in clean accuracy |
| **Feature Squeezing** | Color bit-depth reduction / Spatial smoothing | Strips high-frequency adversarial noise | Degrades classification fidelity on edge cases |
| **Differential Privacy (DP-SGD)**| Clips per-sample gradients & injects calibrated Gaussian noise | Mathematically bounds membership inference risks | Reduces model accuracy on rare attack patterns |
| **Model Watermarking** | Embeds intentional cryptographic backdoors in weights | Proves IP ownership of stolen/extracted models | Does not prevent inference evasion |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Black-Box Transferability of Adversarial Examples
- **Critical Phenomenon:** An attacker does not need access to the target enterprise's proprietary EDR neural network to evade it.
- **The Transferability Property:** Adversarial examples generated against a local, open-source surrogate model (e.g., a standard Random Forest or ResNet trained on public EMBER malware data) frequently **transfer successfully** to fool completely independent commercial EDR products.
- **Why It Happens:** Independent models trained on similar cyber telemetry learn similar decision hyperplanes in feature space.

### 2. Supply-Chain Training Set Poisoning
- **Vulnerability:** Many security teams fine-tune open-source models using public threat datasets downloaded from Hugging Face or public malware feeds.
- **Exploitation:** Malicious actors deliberately submit subtly backdoored samples to public repositories, compromising downstream enterprise fine-tuned models.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Adversarial Machine Learning explores threats targeting the mathematical integrity of models across training, inference, and deployment. At inference time, Evasion Attacks use gradient-based methods like FGSM and PGD to generate adversarial perturbations; in cybersecurity, attackers append optimized bytes to PE file overlay sections to evade neural malware classifiers like MalConv without breaking execution. At training time, Poisoning and Backdoor attacks inject subtle triggers into datasets to create silent bypass backdoors. Furthermore, Black-Box Transferability allows adversaries to train attacks against local open-source models that successfully transfer to bypass commercial proprietary EDRs. Defending against AML requires Adversarial Training—formulated as a Min-Max robust optimization game where models are trained directly on PGD-perturbed samples—combined with gradient clipping and differential privacy."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming black-box models are safe from evasion because weights are private. *Correction:* Due to the Transferability Property, adversaries craft adversarial samples on local surrogate models that successfully evade black-box APIs.
- **Trap 2:** Confusing Poisoning with Evasion. *Correction:* Poisoning corrupts data at *training time* to degrade or backdoor the model; Evasion crafts inputs at *inference time* to deceive an already-trained frozen model.
- **Trap 3:** Applying raw pixel-style perturbations directly to binary executables. *Correction:* Naively flipping bytes in code breaks the binary; cyber evasion attacks must restrict perturbations to non-executable regions (overlay space, slack space, dead-code instructions).

### Expected Follow-Up Questions
1. *What is Adversarial Training and why is it formulated as a Min-Max game?*
   - It is formulated as $\min_\theta \max_{\delta} \mathcal{L}$: the inner maximization finds the worst-case adversarial perturbation that maximizes model loss, while the outer minimization optimizes model weights $\theta$ to minimize that worst-case loss.
2. *How does Differential Privacy (DP-SGD) protect models from data leakage?*
   - By clipping individual sample gradients to a maximum norm and injecting calibrated Gaussian noise during parameter updates, mathematically ensuring that the presence or absence of any single data sample cannot meaningfully alter the final model weights.
