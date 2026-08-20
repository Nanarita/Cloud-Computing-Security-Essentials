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
- AES encrypt/decrypt with MATCH confirmation:  
  <img width="616" height="314" alt="Task1" src="https://github.com/user-attachments/assets/bcb2c1b5-4598-4157-ae70-18ff2916fb1d" />

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
- RSA signature verify output:  
  <img width="727" height="221" alt="Task2" src="https://github.com/user-attachments/assets/f30f07c9-ef5f-469d-891e-9a0ee2470c2d" />

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
  <img width="1901" height="187" alt="Task3(1)" src="https://github.com/user-attachments/assets/fd1d4f9e-d100-4b7d-b2f3-812ccffbc70c" />  
  <img width="1901" height="795" alt="Task3(2)" src="https://github.com/user-attachments/assets/45060b16-77ce-4d54-9016-e362175edfe8" />  
  <img width="1906" height="794" alt="Task3(3)" src="https://github.com/user-attachments/assets/d979f198-af54-4ce6-9ca3-719e41c09a45" />  
  <img width="1908" height="795" alt="Task3(4)" src="https://github.com/user-attachments/assets/51eb023d-ee21-4469-a9f3-364dd36af525" />  
  <img width="1900" height="62" alt="Task3(5)" src="https://github.com/user-attachments/assets/24d6d021-8f27-4de4-b3a8-1a2c97d69fdd" />  

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
- Master Key Creation & Direct KMS Encryption:  
  <img width="992" height="544" alt="Task4" src="https://github.com/user-attachments/assets/2906bc2c-78a7-4309-a0a3-d441e588ba13" />

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
- Data Key Generation:  
  <img width="1659" height="192" alt="Task5 1" src="https://github.com/user-attachments/assets/bd58952d-4bb9-405c-b4c6-be9801f72f96" />  

- Local Encryption via Data Key:  
  <img width="812" height="102" alt="Task5 2" src="https://github.com/user-attachments/assets/bfce0baf-3d1b-4534-9bd4-6e3f521c14eb" />  

- Destruction of Plaintext Key:  
  <img width="534" height="115" alt="Task5 3" src="https://github.com/user-attachments/assets/c774995e-15a8-4b83-b5ff-e0114e3a726e" />  

---

### Task 6 — Per-Tenant Keys & Cryptographic Erasure

Created a second tenant key to show that one tenant's key cannot read another's data.

#### Shell Commands Executed:
```bash
# A separate key for tenant B
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B=<PASTE_KEYID>

# Schedule deletion of tenant A's key (min window)
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7

# Check key state (shows PendingDeletion)
aws $EP kms describe-key --key-id$KEY_A --query 'KeyMetadata.KeyState' --output text

# Cancel deletion and disable it instead to simulate cryptographic erasure state
aws $EP kms cancel-key-deletion --key-id$KEY_A
aws $EP kms describe-key --key-id$KEY_A --query 'KeyMetadata.KeyState' --output text

# Attempt to unwrap tenant A's data key now — it should fail
aws $EP kms decrypt --ciphertext-blob file://datakey.enc 2>&1 | head -3
```

> [!CAUTION]
> Once the key that wrapped the data key is gone, `record.env.enc` is just noise — no one, not even the provider, can decrypt it. This is why per-object/per-tenant keys make deletion provable.

#### Evidence Artifacts:
- Key Deletion & Erasure Testing:  
  <img width="772" height="415" alt="Task6(1)" src="https://github.com/user-attachments/assets/cb24976e-371d-40ea-8b67-509faf963a3b" />  
  <img width="1572" height="444" alt="Task6(2)" src="https://github.com/user-attachments/assets/05180954-47e3-469e-9413-4a59827cb600" />  
  <img width="1139" height="83" alt="Task6(3)" src="https://github.com/user-attachments/assets/4341adc0-e408-4d14-ae08-83666f330746" />  

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
- Hashing and Hash Chain Output:  
  <img width="1138" height="305" alt="Task7" src="https://github.com/user-attachments/assets/6a81ea0d-6a48-40ad-96f1-6345c92021bf" />


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
- **Speed**: Symmetric is lightweight and very fast; asymmetric is mathematically heavy and slow.
- **Key Distribution**: Symmetric requires securely sharing a single secret key beforehand; asymmetric uses a public/private key pair, making distribution safe and easy.
- **Typical Use**: Symmetric is for bulk data encryption (at rest); asymmetric is for key exchange, TLS handshakes, and digital signatures.

#### **Q2. Why is key management described as the weakest link, not the algorithm?**
Modern cryptographic algorithms like AES-256 are mathematically rigorous and completely unfeasible to break through brute force alone. Because the underlying math is rock-solid, attackers completely bypass the cryptography and target human, operational, or architectural flaws such as hardcoded API keys in source code, weak administrative passwords, or misconfigured Key Management Systems. Ultimately, if a key is stolen or mishandled due to poor lifecycle management, the strength of the algorithm becomes entirely meaningless, proving that key security is the true foundation of system defense.

#### **Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.**
- **Envelope Encryption**: Encrypting large files locally using a Data Encryption Key (DEK), then wrapping (encrypting) that DEK with a Master Key.
- **Hardware Protection:** Only the core Master Key needs high-security HSM storage. The wrapped DEK travels safely with the file, saving network overhead since heavy lifting happens locally.

#### **Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?**
In cloud environments, data is dynamically distributed, heavily replicated across multiple availability zones, and stored in invisible snapshots, making traditional physical block overwriting (like dd) unreliable and impossible to verify. Cryptographic erasure bypasses this hardware limitation by ensuring data is always stored as ciphertext. To achieve provable deletion, the tenant simply destroys or disables the Master Key in the KMS. Without that core key, every scattered copy, snapshot, and backup instantly turns into unrecoverable cryptographic noise, guaranteeing instant and verifiable data destruction without needing to wipe physical hardware blocks.

#### **Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?**
A hash chain secures logs by mathematically binding each sequential entry, taking the cryptographic hash of the previous log and feeding it into the calculation for the current one. If a malicious actor attempts to covertly alter any historical log entry, its resulting hash changes completely. Because every subsequent block relies directly on the hash of the one before it, that single modification triggers a cascading domino effect that invalidates every following hash in the chain. This built-in mathematical dependency makes any tampering instantly noticeable upon verification.

---

### 3. Verification Command Output

#### Verification Shell Commands:
```bash
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

#### Evidence Artifact:
- Final Verification Output  
  <img width="843" height="270" alt="3  Verification Command" src="https://github.com/user-attachments/assets/5155b5d3-36ab-45d3-9efb-e132838c2a0c" />  

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
*Report compiled by Student `Siti Nurjannah binti Daud` for IKB42603 Cloud Computing Security Essentials.*
