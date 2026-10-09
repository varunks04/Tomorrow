# Cloud Cryptography, Envelope Encryption, and Object Storage Security

## 1. Topic & Definition
Securing data in cloud computing requires safeguarding assets across two states: **Data in Transit** (enforced via mandatory TLS 1.3 encryption) and **Data at Rest** (enforced via Hardware Security Modules and symmetric encryption).

Because encrypting multi-gigabyte files directly across network APIs introduces latency bottlenecks and bandwidth saturation, cloud providers implement **Envelope Encryption**—a two-tiered cryptographic hierarchy decoupling master key governance from high-speed local data encryption.

---

## 2. How It Works: The Envelope Encryption Architecture

```
+---------------------------------------------------------------------------------------------------+
|                              The Envelope Encryption Lifecycle                                    |
+---------------------------------------------------------------------------------------------------+

  [ STEP 1: KEY GENERATION ]
  Application Server ---> Calls KMS API: `kms:GenerateDataKey(KeyId="KMS-Master-Key")`
                     <--- KMS HSM returns TWO items:
                          1. Plaintext Data Encryption Key (DEK)  [ 256-bit AES key ]
                          2. Encrypted Data Encryption Key (E-DEK)[ Encrypted by KMS Master Key ]

  [ STEP 2: LOCAL ENCRYPTION ]
  Application Server ---> Uses Plaintext DEK to encrypt 10 GB file locally using AES-256-GCM.
                     ---> Instantly PURGES & ZEROIZES Plaintext DEK from RAM!

  [ STEP 3: STORAGE ]
  Application Server ---> Stores Encrypted File + Encrypted DEK side-by-side in Amazon S3.

  -------------------------------------------------------------------------------------------------

  [ DECRYPTION CYCLE ]
  Application Server ---> Fetches Encrypted File + Encrypted DEK from S3.
                     ---> Calls KMS API: `kms:Decrypt(CiphertextBlob=E-DEK)`
                     <--- KMS decrypts inside HSM and returns Plaintext DEK.
                     ---> Server decrypts file in RAM; zeroes DEK immediately upon completion.
```

### Why Envelope Encryption is Mathematically Superior:
1. **Network Performance:** The cloud KMS API only transfers a tiny 32-byte key over the network; massive multi-gigabyte data files are encrypted locally at wire speed.
2. **Master Key Confidentiality:** The KMS Master Key **NEVER leaves the FIPS 140-2 Level 3 validated Hardware Security Module (HSM)** under any circumstances.
3. **Cryptographic Erasure:** Disabling or deleting the KMS Master Key instantly renders petabytes of encrypted data, EBS snapshots, and S3 backups **permanently unrecoverable**, providing instant regulatory data destruction.

---

## 3. Key Management Service (KMS) Governance

### A. AWS Managed Keys vs Customer Managed Keys (CMKs)
- **AWS Managed Keys (`aws/s3`, `aws/ebs`):** Free, created automatically by AWS. Key rotation is fixed at 1 year. Cannot change key policies, cannot audit access independently, and cannot be used across different accounts.
- **Customer Managed Keys (CMKs):** Created and controlled by the enterprise. Full control over KMS Key Policies, cryptographic grants, manual/automatic rotation, and granular CloudTrail audit logging.

### B. The KMS Key Policy Isolation Principle
In AWS IAM, having `AdministratorAccess` does not automatically grant access to KMS keys. **Access to a KMS key is governed strictly by the KMS Key Policy**. If the key policy does not explicitly grant permissions to an IAM identity (or delegate authority to the account root), even the account administrator cannot decrypt data with that key.

---

## 4. Amazon S3 Storage Hardening & Ransomware Defense

```
+---------------------------------------------------------------------------------------------------+
|                            Amazon S3 Multi-Layer Security Architecture                            |
+---------------------------------------------------------------------------------------------------+
  LAYER 1: S3 Block Public Access     --> Account-level and Bucket-level kill-switch on public ACLs
  LAYER 2: S3 Bucket Policy           --> Enforces TLS (aws:SecureTransport) & KMS Encryption
  LAYER 3: Server-Side Encryption     --> Mandates SSE-KMS (Customer Managed Key)
  LAYER 4: S3 Object Versioning       --> Preserves historical versions upon delete/overwrite
  LAYER 5: S3 Object Lock (WORM)      --> Write Once, Read Many (Immutable against Ransomware!)
```

### A. Mandating Encryption and TLS via S3 Bucket Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceHTTPSOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::enterprise-confidential-bucket",
        "arn:aws:s3:::enterprise-confidential-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### B. Defeating Ransomware with S3 Object Lock (WORM Architecture)
Adversaries who compromise cloud administrator credentials attempt to encrypt or delete all S3 backups to force ransom payments.
- **S3 Object Lock (Write Once, Read Many — WORM):** Enforces an immutable retention period on objects.
  - **Governance Mode:** Prevents deletion by standard users, but users with explicit `s3:BypassGovernanceRetention` can override.
  - **Compliance Mode (The Ultimate Defense):** **NO ONE—not even the AWS Account Root User or AWS Support—can delete or overwrite the object until the retention period expires**. If an attacker gains full root access and triggers `DeleteObject`, AWS hard-rejects the API call!

---

## 5. Key Differences Matrix: Server-Side Encryption (SSE) Models

| Feature | SSE-S3 | SSE-KMS | SSE-C | Client-Side Encryption |
| :--- | :--- | :--- | :--- | :--- |
| **Key Managed By** | Amazon S3 | AWS KMS (CMK) | Customer provides raw key per HTTP request | Customer application before upload |
| **Key Rotation** | Automatic | Automatic (Every 365 days) | Customer responsibility | Customer responsibility |
| **Audit Trail** | None | **Full CloudTrail Key Access Audit**| None in AWS | External to AWS |
| **Cross-Account Sharing**| Standard S3 permissions | Requires KMS Key Policy grants | Customer manages | Customer manages |
| **Enterprise Standard** | Basic baseline | **Mandatory Enterprise Standard** | Specific banking edge cases | Extreme zero-trust compliance |

---

## 6. Cybersecurity Relevance, Threats & Misconfigurations

### 1. Public S3 Bucket Breaches
- **Root Cause:** Developers mistakenly applying legacy public ACLs (`public-read`) or writing permissive bucket policies with `"Principal": "*"` and `"Action": "s3:GetObject"`.
- **Historic Impact:** Capital One, Accenture, Booz Allen Hamilton—hundreds of millions of records exposed globally through open buckets.
- **Defense:** Enforce **S3 Block Public Access** at the **AWS Account Level**, completely overriding any misconfigured bucket-level or object-level ACLs across all current and future buckets.

### 2. Accidental Cryptographic Loss (Dangling KMS Keys)
- **Problem:** An administrator deletes a Customer Managed Key (CMK) that was used to encrypt production database EBS volumes and S3 data lakes.
- **Consequence:** After the mandatory 7 to 30-day waiting period, the key is permanently destroyed. All petabytes of data become **mathematically unrecoverable garbage**.
- **Defense:** Never delete KMS keys; disable them instead. Set up Amazon EventBridge alerts on `kms:ScheduleKeyDeletion` events.

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Cloud cryptography operates on Envelope Encryption to balance security, performance, and scale. Instead of sending large data files across the network to be encrypted by KMS, the application calls KMS to generate a 256-bit Data Encryption Key (DEK). The data is encrypted locally using the plaintext DEK, which is immediately purged from RAM, while the encrypted DEK is stored alongside the ciphertext in S3. The master key never leaves the FIPS 140-2 Level 3 Hardware Security Module. For cloud storage security, Amazon S3 requires multi-layered defense: enabling Account-Level Block Public Access, enforcing TLS via `aws:SecureTransport` in bucket policies, and deploying S3 Object Lock in Compliance Mode to achieve immutable WORM storage that defeats cloud ransomware."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Storing the Plaintext Data Encryption Key alongside the data. *Correction:* The plaintext DEK must be zeroized in RAM immediately after encryption; only the *encrypted* DEK is stored with the ciphertext.
- **Trap 2:** Assuming S3 Object Lock in Governance Mode stops an attacker with root access. *Correction:* An attacker with root can bypass Governance Mode; Compliance Mode is required to prevent even root accounts from deleting objects.
- **Trap 3:** Confusing SSE-S3 with SSE-KMS. *Correction:* SSE-S3 uses keys managed entirely by S3 without access audit logs; SSE-KMS uses KMS keys, providing detailed CloudTrail audit logs of every single decrypt operation.

### Expected Follow-Up Questions
1. *What is Cryptographic Erasure?*
   - The practice of rendering encrypted data permanently irretrievable by securely deleting the underlying encryption key (KMS key) rather than overwriting the physical storage media, fulfilling data destruction compliance instantly.
2. *Can the AWS Account Root User decrypt an S3 object encrypted with a Customer Managed KMS key by default?*
   - Not automatically. The KMS Key Policy must explicitly delegate administration to the account root; if the key policy restricts usage exclusively to a specific IAM role and omits the root account, root cannot use the key until the policy is updated.
