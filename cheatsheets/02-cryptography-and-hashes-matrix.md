# Cryptography & Hashes Matrix Cheatsheet

> **Cybersecurity Interview Focus:** Understand cryptographic primitives, mathematical mechanics, key sizes, operational modes, vulnerabilities, and real-world application contexts.

---

## 1. Core Primitives Comparison

| Primitive | Mechanism | Key/Output Size | Primary Goal | Classic Examples | Security Vulnerabilities / Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Symmetric Encryption** | Same key for encrypt & decrypt | 128, 192, 256 bits | Confidentiality at high speed | AES, ChaCha20, DES, 3DES | Key distribution problem; ECB mode leaks image patterns. |
| **Asymmetric Encryption** | Public key encrypts, private decrypts | 2048 - 4096 bits (RSA), 256 - 384 bits (ECC) | Secure key exchange, digital signatures | RSA, ECC (ECDSA, Ed25519) | High CPU overhead; vulnerable to quantum Shor's algorithm. |
| **Cryptographic Hashing** | One-way deterministic fixed output | 256, 512 bits | Data integrity verification | SHA-256, SHA-3, MD5, SHA-1 | MD5 & SHA-1 broken by collision attacks; fast hashes bad for passwords. |
| **Password Hashing** | Slow, salted, memory-hard hashing | Variable | Secure credential persistence | Argon2id, bcrypt, PBKDF2, scrypt | Fast hashes (SHA-256) crackable at billions/sec via GPUs. |
| **HMAC** | Hash + Secret Key | Matches base hash (e.g. 256 bits) | Integrity + Authenticity | HMAC-SHA256 | Prevents hash extension attacks; requires secret key. |
| **Key Exchange** | Shared secret over public channel | Discrete log / Elliptic curves | Session key establishment | Diffie-Hellman, ECDHE | Vulnerable to MITM unless authenticated via PKI. |

---

## 2. Encryption vs Hashing vs Encoding vs Tokenization

| Term | Reversible? | Requires Key? | Mathematical Purpose | Typical Use Case |
| :--- | :---: | :---: | :--- | :--- |
| **Encryption** | **Yes** | **Yes** | Transforming plaintext to ciphertext for confidentiality | Securing data at rest (BitLocker) and in transit (TLS) |
| **Hashing** | **No** | **No** | One-way deterministic digest for data integrity | File checksums, code signing verification |
| **Encoding** | **Yes** | **No** | Changing data format for transmission compatibility | Base64, URL encoding, Hex encoding |
| **Tokenization** | **Yes (via vault)**| **No** | Replacing sensitive data with a non-sensitive surrogate | PCI-DSS credit card processing (Stripe tokens) |

> ⚠️ **Interview Warning:** Stating *"Base64 encrypts the password"* or *"We encrypt passwords in the database using SHA-256"* is an instant red flag. Always say: *"Base64 is an encoding scheme, not encryption; passwords must be salted and hashed using memory-hard functions like Argon2id or bcrypt."*

---

## 3. Symmetric Cipher Modes (AES)

| Mode | Type | IV Required? | Integrity Check? | Parallelizable? | Security Verdict |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **ECB** (Electronic Codebook) | Block | No | No | Yes | ❌ **Insecure:** Identical plaintext blocks produce identical ciphertext (Penguin leak). |
| **CBC** (Cipher Block Chaining) | Block | Yes (Random) | No | Decryption only | ⚠️ **Fragile:** Vulnerable to Padding Oracle attacks unless combined with HMAC. |
| **CTR** (Counter) | Stream | Yes (Nonce) | No | Yes | ⚠️ **Fragile:** Keystream reuse completely destroys confidentiality. |
| **GCM** (Galois/Counter Mode) | AEAD | Yes (Unique Nonce) | **Yes (Built-in Auth Tag)** | Yes | ✅ **Industry Standard:** Authenticated Encryption with Associated Data (TLS 1.3 default). |

---

## 4. Password Storage Best Practices

```text
[Plaintext Password] + [Unique 128-bit Salt] 
               │
               ▼
    [Argon2id / bcrypt / PBKDF2] ─── (Iterated thousands of rounds / memory-hard)
               │
               ▼
    $argon2id$v=19$m=65536,t=3,p=4$... (Salt + Cost + Hash stored in DB)
```

1. **Argon2id (Winner of Password Hashing Competition):** Memory-hard; resists both GPU and ASIC brute force attacks.
2. **bcrypt:** Work factor parameter `cost` (typically 12-14); automatically generates unique per-user salts.
3. **Never use plain SHA-256/MD5:** Modern RTX 4090 GPUs compute over 20 billion SHA-256 hashes per second.
