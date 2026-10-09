# Cryptographic Hash Functions, Salting, Peppering & Key Stretching

> **Domain:** Cybersecurity Fundamentals  
> **Sub-Domain:** Cryptography & Credential Security  
> **Interview Importance:** Critical / Mandatory Technical Interview Assessment  

---

## 1. Topic & Definitions

- **Cryptographic Hash Function:** A mathematical algorithm that takes an arbitrary-length input (message) and transforms it into a fixed-length string of bits (hash digest or checksum) in a strictly one-way, deterministic fashion.
- **The Avalanche Effect:** A crucial property where any microscopic change in the input (e.g. flipping a single bit from `0` to `1`) results in a radical, unpredictable change in more than 50% of the output hash bits.
- **Salt:** A cryptographically random, unique sequence of bits (minimum 128 bits) generated per-user and combined with a plaintext password prior to hashing.
- **Pepper:** A secret cryptographic key stored separately from the database (e.g. in a hardware security module / HSM or environment variable) that is combined with all passwords prior to hashing.
- **Key Stretching (Slow Hashing):** A technique that deliberately inserts computational iterations and memory allocation requirements into the hashing algorithm, making brute-force dictionary attacks mathematically infeasible.

---

## 2. The 5 Mathematical Properties of a Cryptographic Hash

```text
+─────────────────────────────────────────────────────────────────────────────+
|                 THE 5 PROPERTIES OF A SECURE HASH FUNCTION                  |
+─────────────────────────────────────────────────────────────────────────────+
| 1. DETERMINISTIC:                                                           |
|    • The exact same input string ALWAYS produces the identical hash digest. |
|─────────────────────────────────────────────────────────────────────────────|
| 2. QUICK COMPUTATION:                                                       |
|    • Computing the hash of a file or message is fast for legitimate uses.   |
|─────────────────────────────────────────────────────────────────────────────|
| 3. PRE-IMAGE RESISTANCE (One-Way Property):                                 |
|    • Given a hash digest H, it is computationally impossible to find any    |
|      original message M such that Hash(M) = H.                              |
|─────────────────────────────────────────────────────────────────────────────|
| 4. SECOND PRE-IMAGE RESISTANCE (Weak Collision Resistance):                 |
|    • Given an input M1, it is computationally impossible to find a          |
|      DIFFERENT input M2 such that Hash(M1) = Hash(M2).                      |
|─────────────────────────────────────────────────────────────────────────────|
| 5. COLLISION RESISTANCE (Strong Collision Resistance):                      |
|    • It is computationally impossible to find ANY two arbitrary messages     |
|      M1 and M2 such that Hash(M1) = Hash(M2).                               |
|    • Evaluated via the Birthday Paradox: Finding a collision in an n-bit    |
|      hash requires only 2^(n/2) operations.                                 |
+─────────────────────────────────────────────────────────────────────────────+
```

### Hash Algorithm Status Matrix

| Algorithm | Digest Size | Security Status | Known Attack & Vulnerability |
| :--- | :---: | :---: | :--- |
| **MD5** | 128 bits | ❌ **BROKEN** | Trivial collisions generated in sub-seconds on laptops. |
| **SHA-1** | 160 bits | ❌ **BROKEN** | Broken by Google's **SHAttered** attack (2017) finding PDF collision. |
| **SHA-256 / SHA-512**| 256 / 512 bits | ✅ **SECURE** | Industry standard for file integrity and digital signatures. |
| **SHA-3 (Keccak)** | Variable | ✅ **SECURE** | Sponge construction; structurally distinct from SHA-2. |

---

## 3. Why Fast Hashes (SHA-256) are DANGEROUS for Passwords!

```text
The Core Paradox:
Fast hashes are designed to verify gigabyte files in milliseconds.
BUT THAT SPEED DESTROYS PASSWORDS!
```

- A modern consumer graphics card (e.g. NVIDIA RTX 4090) computes over **25 BILLION SHA-256 hashes per second** using Hashcat.
- If a database containing SHA-256 password hashes leaks, an attacker brute-forces every 8-character password across all upper, lower, numbers, and symbols ($95^8 \approx 6.6 \times 10^{15}$ combinations) in **less than 3 days**!
- **Conclusion:** **NEVER use plain SHA-256, SHA-512, or MD5 for password storage.**

---

## 4. The Modern Password Hashing Architecture

```text
[ Plaintext Password ] + [ Unique 128-bit Salt ] + [ Secret Pepper (in HSM) ]
                                  │
                                  ▼
                [ MEMORY-HARD SLOW HASH ALGORITHM ]
              (Argon2id / bcrypt / PBKDF2 / scrypt)
                                  │
                                  ▼
$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ...$w9/V3X...
└──┬───┘  └──┬─┘ └──────┬─────┘ └─────┬────┘ └───┬───┘
   │         │          │             │          └── Final Hash Digest
   │         │          │             └───────────── Salt (Base64 encoded)
   │         │          └─────────────────────────── Cost: Memory (64MB), Iterations (3), Threads (4)
   │         └────────────────────────────────────── Algorithm Version
   └──────────────────────────────────────────────── Algorithm Identifier
```

### 1. Salting & Defeating Rainbow Tables
- **Rainbow Table:** A massive precomputed lookup table mapping plaintext passwords to their resulting hash digests.
- **How Salting Defeats It:** Because every user receives a unique, random salt (e.g. `Salt_Alice` vs `Salt_Bob`), identical passwords (`Password123`) produce completely different hash strings in the database. Attackers cannot use precomputed tables and must crack each account individually from scratch.

### 2. Password Hashing Algorithms Evaluated

| Algorithm | Key Mechanism & Strengths | Memory-Hard? | Modern Security Verdict |
| :--- | :--- | :---: | :--- |
| **Argon2id** | Winner of the Password Hashing Competition (2015). Combines Argon2d (resists GPU cracking) and Argon2i (resists side-channel attacks). | ✅ **YES** (Configurable RAM consumption) | 🏆 **Gold Standard:** Highest recommended algorithm by OWASP. |
| **bcrypt** | Based on the Eksblowfish block cipher. Configurable cost factor ($2^{\text{cost}}$ rounds; standard is 12-14). Automatically generates salts. | ❌ No | ✅ **Secure & Widely Supported:** Excellent legacy and enterprise choice. |
| **PBKDF2** | Applies HMAC in a loop thousands of times (NIST recommends $\ge 600,000$ iterations for SHA-256). | ❌ No | ⚠️ **Acceptable for Compliance:** Resists CPU, but vulnerable to fast GPU/ASIC parallelization due to low RAM usage. |

---

## 5. Salt vs. Pepper Comparison

| Dimension | Salt | Pepper |
| :--- | :--- | :--- |
| **Uniqueness** | **Unique per user** (Randomly generated during signup). | **Global or shared** across the application. |
| **Storage Location**| Stored openly in the **database table** alongside the hash. | Stored **outside the database** (AWS KMS, HashiCorp Vault, HSM, env). |
| **Secret Status** | Public knowledge; does not need to be hidden. | **Strictly Secret**; if compromised, pepper value is lost. |
| **Primary Protection**| Defeats Rainbow Tables and cross-user deduplication. | Protects database dumps: if SQLi dumps the DB, hashes cannot be cracked without the HSM pepper! |

---

## 6. Interview Takeaway: What to Say Under Pressure

### 🎙️ 60-Second Elevator Pitch
> *"Cryptographic hash functions provide one-way deterministic data integrity through pre-image resistance and collision resistance. While fast algorithms like SHA-256 are ideal for file checksums and digital signatures, they are catastrophically insecure for password storage because modern GPUs compute tens of billions of SHA-256 hashes per second. Secure password storage mandates slow, memory-hard key stretching algorithms—primarily Argon2id (the OWASP gold standard) or bcrypt. Passwords must always be combined with a unique, cryptographically random 128-bit Salt stored in the database to defeat Rainbow Tables, and optionally a secret Pepper stored in a hardware security module (HSM) to ensure database dumps cannot be cracked offline."*

### ⚠️ Common Traps & Gotchas
- **Trap:** Thinking the Salt must be kept secret.  
  *Correction:* The Salt does NOT need to be secret! It is stored in plaintext directly in the database row next to the hash. Its sole purpose is to inject unique randomness to ensure two identical passwords never generate the same hash, rendering precomputed Rainbow Tables useless.
- **Trap:** Recommending MD5 or SHA-1 for any new security use case.  
  *Correction:* Both MD5 and SHA-1 have broken collision resistance. Always recommend SHA-256/SHA-512 for data integrity, and Argon2id for password credential persistence.
