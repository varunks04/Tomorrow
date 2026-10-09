# CNNs, RNNs, and Sequence Modeling for Threat Detection

## 1. Topic & Definition
Specialized deep learning architectures process unstructured and sequential cyber telemetry beyond flat tabular logs:

1. **Convolutional Neural Networks (CNNs):** Neural networks that utilize sliding convolutional kernels (filters) to learn shift-invariant local spatial patterns. In cybersecurity, CNNs analyze raw packet payloads as 1D byte sequences and binary executables transformed into 2D grayscale imagery.
2. **Recurrent Neural Networks (RNNs, LSTMs, GRUs):** Networks with internal feedback loops and persistent memory states designed to model temporal dependencies and sequential structures (system call sequences, shell command executions, C2 beacon intervals).

---

## 2. How It Works: Architectural Mechanics and Computations

### A. Convolutional Neural Networks (CNNs)

```
[ Raw Executable PE File / PCAP Bytes ]
                |
                v  (Reshaped into 2D Byte Grid: 0-255 pixels)
       [ 2D Grayscale Image ]
                |
                v  [ Convolutional Layer (Kernels sliding across byte window) ]
       [ Feature Maps ]
                |
                v  [ Max Pooling (Spatial downsampling / dimension reduction) ]
       [ Pooled Representations ]
                |
                v  [ Dense Fully Connected Layer + Softmax ]
       [ Output: Malware Family Classification (Trojan, Ransomware, Benign) ]
```

#### Core CNN Operations
1. **2D Convolution:** Slides a small weight matrix (Kernel $K \in \mathbb{R}^{k \times k}$) over the input matrix $X$:
   $$S(i, j) = (X * K)(i, j) = \sum_{m} \sum_{n} X(i - m, j - n) K(m, n)$$
2. **Pooling (Max Pooling):** Selects the maximum activation in a local window, ensuring **spatial translation invariance** (a malicious shellcode shell loop is detected regardless of its exact memory offset in the PE file).

#### Practical Malware Image Classification (The Malimg Paradigm)
- An unparsed binary file is read as an array of 8-bit unsigned integers ($0 - 255$).
- The 1D array is reshaped into a 2D matrix of fixed width (e.g., 256, 512, or 1024 columns depending on file size).
- The matrix is visualized as a grayscale image:
  - `.text` (executable code): Smooth, repetitive, fine-grained texture.
  - `.rdata` (read-only data): Medium-density textures.
  - `.data` (variables): Sparse, white/black blocks.
  - Encrypted / Compressed Payload: Uniform, noisy, high-entropy "television static" pattern.
- CNNs classify these image textures into known malware families (e.g., WannaCry, Emotet) in milliseconds without disassembling code or unpacking the binary!

---

### B. Recurrent Neural Networks (RNN, LSTM, and GRU)

```
       +-------------------------------------------------------------+
       |             Long Short-Term Memory (LSTM) Cell              |
       |                                                             |
       |   Cell State C_{t-1} ---------------------[+]-------------> C_t
       |                           |                ^                |
       |                           |                | (*)            |
       |                           v                |                v
       |                    [ Forget Gate ]  [ Input Gate ]   [ Output Gate ]
       |                           f_t              i_t              o_t
       |                           |                |                |
       |   Hidden State h_{t-1} ---+----------------+----------------+--> h_t
       |                           |                                 |
       |   Input x_t --------------+---------------------------------+
       +-------------------------------------------------------------+
```

#### 1. The Long Short-Term Memory (LSTM) Equations
Standard RNNs suffer from vanishing gradients when tracking sequences longer than 10–20 steps. LSTMs solve this via an explicit linear **Cell State ($C_t$)** regulated by three multiplicative gates:

- **Forget Gate ($f_t$):** Decides what prior sequence history to discard:
  $$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$
- **Input Gate ($i_t$ & $\tilde{C}_t$):** Decides what new information to store in the cell state:
  $$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i), \quad \tilde{C}_t = \tanh(W_c \cdot [h_{t-1}, x_t] + b_c)$$
- **Cell State Update:**
  $$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$$
- **Output Gate ($o_t$ & $h_t$):** Decides what filtered state to output:
  $$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o), \quad h_t = o_t \odot \tanh(C_t)$$

#### 2. Gated Recurrent Unit (GRU)
A computationally streamlined variant that merges the cell state and hidden state, using only two gates: **Reset Gate ($r_t$)** and **Update Gate ($z_t$)**. Offers 25–30% faster training on large log volumes with comparable accuracy.

---

## 3. Practical Cybersecurity Application Mapping

### 1. Host Intrusion Detection via System Call Modeling (HIDS)
- **Data Source:** Linux `auditd` or `strace` recording sequences of kernel system calls:
  `[ openat, read, mprotect, ptrace, socket, connect, write ]`
- **LSTM Modeling:** The network learns the transition probability distribution of legitimate application system call sequences.
- **Detection:** When an adversary executes an in-memory buffer overflow or code injection, the unexpected system call transition sequence (e.g., sudden `mprotect` followed by `ptrace`) triggers a massive sequence perplexity spike, flagging an anomaly before persistence is established.

### 2. Command-Line / PowerShell Script Obfuscation Detection
- Adversaries obfuscate malicious commands using environment variable insertion, ticks, and variable concatenation:
  `powershell.exe -w hidden -enc JABzAD0ATgBlAHcALQBPAGIA...`
- LSTMs and GRUs treat command strings as character or token sequences, classifying whether the sequence exhibits adversarial obfuscation signatures regardless of variable renaming.

---

## 4. Key Differences Matrix: CNN vs LSTM vs GRU

| Dimension | 1D / 2D CNN | LSTM | GRU |
| :--- | :--- | :--- | :--- |
| **Input Structure** | Grayscale byte grids or fixed packet windows | Sequential token / event streams | Sequential token / event streams |
| **Temporal Context** | Local receptive field (n-gram patches) | Long-range context ($10^2 - 10^3$ steps) | Moderate context ($10^2$ steps) |
| **Parallel Training**| **High** (Convolutions run in parallel on GPU) | **Low** (Step $t$ strictly depends on $t-1$) | **Low** (Step $t$ depends on $t-1$) |
| **Parameter Count** | Low to Moderate | High (4 weight matrices per cell) | Moderate (3 weight matrices per cell) |
| **Primary Cyber Role**| Malware binary classification, PCAP payloads | System call anomaly detection, C2 beaconing | Fast stream analysis, DNS tunneling |

---

## 5. Cybersecurity Relevance, Threats & Attack Vectors

### 1. Adversarial Perturbation Attacks on CNN Malware Classifiers
- **Mechanism:** Fast Gradient Sign Method (FGSM) or Jacobian-based Saliency Map Attacks (JSMA).
- **Execution:** An attacker computes the gradient of the CNN loss with respect to the input binary bytes. They add a tiny, carefully crafted perturbation into non-functional sections of the PE header or the end of the file (overlay padding).
- **Impact:** The visual texture shifts subtly in feature space, flipping the CNN classification from "Ransomware" to "Benign Calculator" without corrupting the malware's malicious execution capability!
- **Defense:** Adversarial training (injecting perturbed malware variants during training) and stripping unused overlay sections prior to feature extraction.

### 2. Mimicry Attacks Against Sequence Models (HIDS Evasion)
- **Mechanism:** An attacker who knows an LSTM is monitoring system call sequences pads their malicious system calls with benign "dummy" calls (e.g., interleaving `getpid()` or harmless file reads) to reset the LSTM hidden state and blend into normal execution profiles.
- **Defense:** Combine sequence modeling with context-aware argument analysis (inspecting paths and buffer arguments, not just call names).

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Deep learning excels at extracting representations from complex, unstructured cybersecurity telemetry. Convolutional Neural Networks (CNNs) process spatial and structural relationships; by converting raw binary executables into 2D grayscale images (the Malimg approach), CNNs classify malware families based on visual code textures—such as encrypted payload noise—without disassembly or dynamic execution. For sequential cyber data, Recurrent Neural Networks, specifically LSTMs and GRUs, solve vanishing gradients through gated memory cells. LSTMs model temporal dependencies across system call sequences to detect in-memory exploitation and evaluate packet inter-arrival times to uncover hidden C2 beaconing. However, CNNs are vulnerable to adversarial byte perturbations, and LSTMs can be targeted by mimicry padding attacks."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming CNNs can only process images. *Correction:* 1D CNNs are widely used on raw packet payload bytes, log character streams, and assembly opcode sequences.
- **Trap 2:** Confusing the LSTM Forget Gate with the Reset Gate. *Correction:* The Forget Gate belongs to LSTMs (controls how much of the prior cell state $C_{t-1}$ is retained); the Reset Gate belongs to GRUs (controls how much of the previous hidden state is mixed with candidate state).
- **Trap 3:** Expecting LSTMs to train as fast as CNNs. *Correction:* LSTMs must compute sequentially step-by-step ($t$ requires $t-1$), preventing massive parallel GPU acceleration unlike CNN convolutions and Transformer self-attention.

### Expected Follow-Up Questions
1. *Why does visualizing a malware binary as an image allow CNNs to identify packed code?*
   - Packed or encrypted code sections exhibit near-maximum Shannon entropy, appearing visually as dense, uniformly distributed random noise, contrasting sharply with the structured, repetitive textures of unencrypted x86 machine instructions.
2. *What is Backpropagation Through Time (BPTT)?*
   - BPTT unrolls the recurrent network across all time steps, calculating gradients of the loss with respect to shared weights by summing partial derivatives across the entire temporal sequence.
