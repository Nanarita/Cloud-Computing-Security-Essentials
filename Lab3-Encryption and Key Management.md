# IKB42603 Cloud Computing Security Essentials
## Lab 3 Comprehensive Report: Data Protection: Encryption & Key Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Lab Assignment:** Lab 3 (Weeks 5–6) — Data Protection: Encryption & Key Management  
**Course Learning Outcome:** CLO2 — Construct secure cloud operations that safeguard data integrity (VBE3)  
**Student Name:** `Siti Nurjannah binti Daud`  
**Student ID:** `52215124446`  

---

## Executive Summary & Learning Outcomes

This step-by-step lab report documents the implementation and evaluation of cryptography and key management techniques to protect data at rest and in transit.

### Key Objectives & Achievements:
1. **Symmetric & Asymmetric Encryption (Session A - Week 5)**: Encrypted and decrypted data using symmetric (AES) and asymmetric (RSA) cryptography.
2. **Encryption in Transit (TLS)**: Protected data in transit with TLS using a self-signed certificate and observed the encrypted communication channel.
3. **Key Management Service (KMS) & Envelope Encryption (Session B - Week 6)**: Used a KMS (LocalStack) to generate a Master Key and Data Keys, successfully implementing envelope encryption for local data.
4. **Per-Tenant Keys & Cryptographic Erasure**: Applied per-tenant keys for strict data isolation and performed cryptographic erasure by scheduling key deletion, rendering data permanently unrecoverable.
5. **Integrity & Tamper-Evidence**: Verified data integrity with hashing algorithms (SHA-256) and built a simple hash chain to produce a tamper-evident record.

---

## Table of Contents
1. [Session A (Week 5) — Encryption Fundamentals](#session-a-week-5--encryption-fundamentals)
   - [Task 1 — Symmetric Encryption (Data at Rest)](#task-1--symmetric-encryption-data-at-rest)
   - [Task 2 — Asymmetric Encryption & Digital Signatures](#task-2--asymmetric-encryption--digital-signatures)
   - [Task 3 — Encryption in Transit (TLS)](#task-3--encryption-in-transit-tls)
2. [Session B (Week 6) — Key Management, Envelope Encryption & Erasure](#session-b-week-6--key-management-envelope-encryption--erasure)
   - [Task 4 — Create and Use a KMS Master Key](#task-4--create-and-use-a-kms-master-key)
   - [Task 5 — Envelope Encryption](#task-5--envelope-encryption)
   - [Task 6 — Per-Tenant Keys & Cryptographic Erasure](#task-6--per-tenant-keys--cryptographic-erasure)
   - [Task 7 — Integrity & Tamper-Evidence](#task-7--integrity--tamper-evidence)
3. [Deliverables & Assessment](#deliverables--assessment)
   - [1. Screenshots Reference Checklist](#1-screenshots-reference-checklist)
   - [2. Short-Answer Questions (Q1 - Q5)](#2-short-answer-questions-q1---q5)
   - [3. Verification Command Output](#3-verification-command-output)
   - [4. Security Best-Practices Checklist](#4-security-best-practices-checklist)
   - [5. Cleanup & Teardown](#5-cleanup--teardown)
   - [6. Expansion Ideas (Advanced Students)](#6-expansion-ideas-advanced-students)

---

## Session A (Week 5) — Encryption Fundamentals

### Task 1 — Symmetric Encryption (Data at Rest)

Created a sensitive file and encrypted it with AES-256. Then decrypted it. One shared key does both — fast, but the key must be protected.

#### Shell Commands Executed:
```bash
# Create a sample sensitive record
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt

# Encrypt with AES-256 (you will be prompted for a passphrase = the key)
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc

# Prove it is unreadable
cat record.enc

# Decrypt back
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

#### Terminal Execution & Output:
```text
enter aes-256-cbc encryption password:
Verifying - enter aes-256-cbc encryption password:
UWe|T(+^PgO
                           %
enter aes-256-cbc decryption password:
MATCH: decryption successful
```

#### Evidence Artifact:
- AES encrypt/decrypt with MATCH confirmation: ![Task1.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task1.PNG)

---

### Task 2 — Asymmetric Encryption & Digital Signatures

Generated an RSA key pair. Anyone can encrypt with the public key; only the private key decrypts. Signatures prove origin and integrity.

#### Shell Commands Executed:
```bash
# Generate a 2048-bit key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# Encrypt with the PUBLIC key, decrypt with the PRIVATE key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt

# Sign with the PRIVATE key; verify with the PUBLIC key
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

#### Terminal Execution & Output:
```text
Generating RSA private key, 2048 bit long modulus (2 primes)
...
writing RSA key
Verified OK
```

> [!NOTE]
> Note how the roles reverse: encryption uses the public key, signing uses the private key. This is the basis of PKI and TLS.

#### Evidence Artifact:
- RSA signature verify output: ![Task2.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task2.PNG)

---

### Task 3 — Encryption in Transit (TLS)

Served a file over HTTPS with a self-signed certificate and confirmed the channel is encrypted.

#### Shell Commands Executed:
```bash
# Generate a self-signed certificate
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
 -days 7 -nodes -subj '/CN=localhost'

# Serve HTTPS on port 8443 using a small container
docker run --rm -d --name tls -p 8443:443 \
 -v $(pwd)/cert.pem:/etc/nginx/cert.pem -v $(pwd)/key.pem:/etc/nginx/key.pem \
 -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx

# Connect over TLS (-k accepts the self-signed cert)
curl -k https://localhost:8443/record.txt
```

#### Terminal Execution & Output:
```text
Generating a RSA private key
.....+++++
.....+++++
writing new private key to 'key.pem'
-----
8b2f8a...
Patient: Ahmad, Diagnosis: confidential
```

> [!TIP]
> **Security tip**: Compare mentally with plain HTTP: over HTTP the record would travel in clear text and any on-path attacker could read it (eavesdropping, Week 3). TLS makes intercepted traffic unreadable.

#### Evidence Artifacts:
- TLS Setup & Connection Tests: 
  - ![Task3(1).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(1).PNG)
  - ![Task3(2).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(2).PNG)
  - ![Task3(3).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(3).PNG)
  - ![Task3(4).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(4).PNG)
  - ![Task3(5).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(5).PNG)

---

## Session B (Week 6) — Key Management, Envelope Encryption & Erasure

### Task 4 — Create and Use a KMS Master Key

Created a Customer Master Key (CMK) in LocalStack KMS to act as the primary encryption key.

#### Shell Commands Executed:
```bash
EP='--endpoint-url=http://localhost:4566'

# Create a customer master key (CMK) and capture its KeyId
aws $EP kms create-key --description 'CCSE tenant-A master key'

# Copy the KeyId from the output into KEY_A below
KEY_A=<PASTE_KEYID>

# Encrypt a small secret directly with KMS
aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
 --query CiphertextBlob --output text
```

#### Evidence Artifact:
- Master Key Creation & Direct KMS Encryption: ![Task4.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task4.PNG)

---

### Task 5 — Envelope Encryption

For large data, it is not encrypted with the master key directly. Instead, a Data Key is generated, the data is encrypted locally with the Data Key, and the Data Key itself is wrapped by the Master Key.

#### Shell Commands Executed:
```bash
# 5.1 Ask KMS for a data key (returns plaintext + encrypted versions)
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
 --query '[Plaintext,CiphertextBlob]' --output text
# Save column 1 as datakey.b64 (plaintext) and column 2 as datakey.enc (wrapped)

# 5.2 Encrypt the big file locally with the PLAINTEXT data key
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc \
 -pass file:./datakey.bin

# 5.3 Destroy the plaintext data key from disk — keep only the wrapped copy
rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

> [!NOTE]
> To read the data later you send `datakey.enc` back to KMS (`kms decrypt`) to unwrap it, use it, then discard it. Only the small master key ever needs hardware-grade protection.

#### Evidence Artifacts:
- Data Key Generation: ![Task5.1.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task5.1.PNG)
- Local Encryption via Data Key: ![Task5.2.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task5.2.PNG)
- Destruction of Plaintext Key: ![Task5.3.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task5.3.PNG)

---

### Task 6 — Per-Tenant Keys & Cryptographic Erasure

Created a second tenant key to show that one tenant's key cannot read another's data. Scheduled key deletion to simulate cryptographic erasure.

#### Shell Commands Executed:
```bash
# A separate key for tenant B
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B=<PASTE_KEYID>

# Schedule deletion of tenant A's key (min window)
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7

# Disable it immediately to simulate erasure
aws $EP kms disable-key --key-id $KEY_A

# Attempt to unwrap tenant A's data key now — it should FAIL
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

#### Terminal Execution & Output:
```text
{
    "KeyId": "arn:aws:kms:us-east-1:000000000000:key/...",
    "DeletionDate": "..."
}
An error occurred (KMSInvalidStateException) when calling the Decrypt operation: arn:aws:kms:us-east-1:000000000000:key/... is pending deletion.
```

> [!CAUTION]
> Once the key that wrapped the data key is gone, `record.env.enc` is just noise — no one, not even the provider, can decrypt it. This is why per-object/per-tenant keys make deletion provable.

#### Evidence Artifacts:
- Key Deletion & Erasure Testing:
  - ![Task6(1).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task6(1).PNG)
  - ![Task6(2).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task6(2).PNG)
  - ![Task6(3).PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task6(3).PNG)

---

### Task 7 — Integrity & Tamper-Evidence

Encryption protects confidentiality; hashing protects integrity. Detected tampering and built a simple hash chain.

#### Shell Commands Executed:
```bash
# Fingerprint the file
sha256sum record.txt

# Tamper with a copy and show the hash changes
cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt

# Hash chain: each entry includes the previous hash (tamper-evident log)
PREV=0
for line in 'login ok' 'file read' 'export data'; do \
 PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
 echo "$line | $PREV"; done
```

#### Evidence Artifact:
- Hashing and Hash Chain Output: ![Task7.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task7.PNG)

---

## Deliverables & Assessment

### 1. Screenshots Reference Checklist

| Task | Description | Evidence File Path | Status |
|---|---|---|---|
| **Task 1** | AES encrypt/decrypt with the MATCH confirmation | [`Lab3/Task1.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task1.PNG) | Verified |
| **Task 2** | RSA signature verify output showing 'Verified OK' | [`Lab3/Task2.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task2.PNG) | Verified |
| **Task 3** | The `curl -k https://...` output over TLS | [`Lab3/Task3(1).PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task3(1).PNG) to `Task3(5).PNG` | Verified |
| **Task 4** | The KMS KeyId creation | [`Lab3/Task4.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task4.PNG) | Verified |
| **Task 5** | The envelope-encryption steps | [`Lab3/Task5.1.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task5.1.PNG) to `Task5.3.PNG` | Verified |
| **Task 6** | The failed kms decrypt after key erasure | [`Lab3/Task6(1).PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task6(1).PNG) to `Task6(3).PNG` | Verified |
| **Task 7** | The differing SHA-256 hashes and hash chain | [`Lab3/Task7.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/Task7.PNG) | Verified |
| **Verification**| Verification commands output | [`Lab3/3. Verification Command.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/3.%20Verification%20Command.PNG) | Verified |

---

### 2. Short-Answer Questions (Q1 - Q5)

#### **Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.**
- **Speed**: Symmetric encryption is highly performant and fast. Asymmetric encryption is significantly slower and computationally expensive.
- **Key Distribution**: Symmetric encryption relies on a single shared key, which introduces severe key distribution challenges since both parties must securely exchange this key beforehand. Asymmetric encryption uses a public/private key pair, making key distribution trivial as the public key can be openly shared without compromising the private key.
- **Typical Use**: Symmetric encryption is used for bulk data encryption (data at rest, large files). Asymmetric encryption is used for exchanging symmetric keys, establishing TLS handshakes, and digital signatures.

#### **Q2. Why is key management described as the weakest link, not the algorithm?**
Modern encryption algorithms like AES-256 are mathematically proven and practically unbreakable by brute force. Instead of attacking the cryptography, attackers target human and operational weaknesses: hardcoded keys, leaked passphrases, or poorly secured Key Management Systems. If the key is stolen, the strength of the algorithm is irrelevant. Key Management provides the critical access control and lifecycle policies that actually secure the ciphertext.

#### **Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.**
- **Envelope Encryption**: This is the practice of encrypting plaintext data locally with a generated Data Encryption Key (DEK). To protect the DEK, it is wrapped (encrypted) by a Master Key (Key Encryption Key or KEK). 
- **Hardware-Grade Protection**: Only the small Master Key needs to be stored in an HSM or KMS with hardware-grade security. The wrapped DEK can safely be stored alongside the encrypted data payload. This dramatically reduces KMS API latency and load, as bulk encryption is done locally and only the DEK needs to be transmitted to the KMS for wrapping and unwrapping.

#### **Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?**
In cloud environments, storage is decoupled and highly replicated (SAN, snapshots, multi-AZ backups), making it impossible to guarantee that every physical byte of a file has been overwritten by tools like `dd`. Cryptographic erasure solves this by wrapping the data in encryption. To "delete" the data, the tenant simply destroys the Master Key in the KMS. Without the key, every distributed copy and backup of the data immediately turns into unrecoverable cryptographic noise, ensuring instant and provable data destruction.

#### **Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?**
A hash chain links sequential log entries by taking the cryptographic hash of the previous entry and feeding it as input into the hash calculation of the current entry. If a malicious actor alters any historical log entry, its hash value will change completely. This modified hash will subsequently invalidate the calculation of the next entry, cascading the failure through the entire remainder of the chain. This domino effect makes any tampering instantly evident upon verification of the final hash.

---

### 3. Verification Command Output

#### Verification Shell Commands:
```bash
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

#### Evidence Artifact:
- Final Verification Output: ![3. Verification Command.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab3/3.%20Verification%20Command.PNG)

---

### 4. Security Best-Practices Checklist

- [x] **Data encrypted at rest (AES) and decryption verified.**
- [x] **Asymmetric keys used correctly (encrypt with public, sign with private).**
- [x] **Data protected in transit with TLS.**
- [x] **Envelope encryption used; plaintext data key not left on disk.**
- [x] **Per-tenant keys used; cryptographic erasure demonstrated.**
- [x] **Integrity verified with hashing / hash chain.**

---

### 5. Cleanup & Teardown

To stop the background services and remove the generated certificates and keys.

#### Teardown Commands:
```bash
docker stop tls 2>/dev/null
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt
docker stop localstack && docker rm localstack
```

---

### 6. Expansion Ideas (Advanced Students)

1. **Store the master key in a software HSM (SoftHSM)** and use PKCS#11 to sign — model hardware key protection.
2. **Stand up HashiCorp Vault** in a container and use its transit engine for envelope encryption.
3. **Configure mutual TLS (mTLS)** between two containers so both sides authenticate.
4. **Automate key rotation** and re-wrap existing data keys under a new master key.

---
*Report compiled by Student `fikri` for IKB42603 Cloud Computing Security Essentials.*
