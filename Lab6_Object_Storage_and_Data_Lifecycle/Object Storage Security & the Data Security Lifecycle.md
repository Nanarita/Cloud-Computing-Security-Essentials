# IKB42603 Cloud Computing Security Essentials
## Lab 6 Comprehensive Report: Object Storage Security & the Data Security Lifecycle (Bucket exposure, resource policies, SSE-KMS, versioning and provable deletion)

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Lab Assignment:** Lab 6 (Weeks 11–12) — Object Storage Security & the Data Security Lifecycle  
**Course Learning Outcome:** CLO2 — Construct secure cloud operations that safeguard data confidentiality and integrity (VBE3)  
**Student Name:** `Siti Nurjannah Binti Daud`  
**Student ID:** `52215124446`  

---

## Executive Summary & Learning Outcomes

This step-by-step lab report documents the implementation and evaluation of object storage security controls across the entire data security lifecycle using Amazon S3 on LocalStack. It traces the journey of data from classification and exposure remediation, through encryption at rest and delegated access, to versioning, retention, and provable cryptographic erasure.

### Key Objectives & Achievements:
1. **Data Classification & The Exposure Problem (Session A - Week 11)**: Classified health records into public, internal, and confidential tiers. Demonstrated the archetypal cloud breach via an overly permissive bucket policy (`Principal: "*"`) and remediated it using **Block Public Access** guardrails and least-privilege resource policies.
2. **Identity vs. Resource Policies**: Evaluated the intersection of IAM identity-based policies and S3 resource-based bucket policies, proving that an explicit `Deny` in a resource policy successfully overrides broad IAM `Allow` permissions.
3. **Encryption at Rest & Delegated Access (Session B - Week 12)**: Implemented Server-Side Encryption with KMS (SSE-KMS) as a default bucket behavior. Demonstrated secure, time-bounded sharing using **Presigned URLs** and analyzed the environmental constraints of `aws:SecureTransport` condition keys.
4. **Data Remanence & Cryptographic Erasure**: Explored object-level data remanence caused by versioning and delete markers. Addressed regulatory compliance (PDPA/GDPR) by deploying automated Lifecycle configurations and proving true data destruction via **Cryptographic Erasure** of the KMS key.

---

## Table of Contents
1. [Session A (Week 11): Object Storage & the Exposure Problem](#session-a-week-11-object-storage--the-exposure-problem)
   - [One-Time Environment Setup](#one-time-environment-setup)
   - [Task 1: Classify the Data Before You Store It](#task-1-classify-the-data-before-you-store-it)
   - [Task 2: Reproduce the Archetypal Breach](#task-2-reproduce-the-archetypal-breach)
   - [Task 3: Remediate with Block Public Access](#task-3-remediate-with-block-public-access)
   - [Task 4: Identity Policy vs Resource Policy](#task-4-identity-policy-vs-resource-policy)
2. [Session B (Week 12): Protecting, Retaining and Retiring Data](#session-b-week-12-protecting-retaining-and-retiring-data)
   - [Task 5: Default Encryption at Rest (SSE-KMS)](#task-5-default-encryption-at-rest-sse-kms)
   - [Task 6: Delegated Access and the Condition-Key Trap](#task-6-delegated-access-and-the-condition-key-trap)
   - [Task 7: Versioning, Delete Markers & Data Remanence](#task-7-versioning-delete-markers--data-remanence)
   - [Task 8: Lifecycle, Retention & Cryptographic Erasure](#task-8-lifecycle-retention--cryptographic-erasure)
3. [Deliverables & Assessment](#deliverables--assessment)
   - [1. Screenshots Reference Checklist](#1-screenshots-reference-checklist)
   - [2. Data Classification Table](#2-data-classification-table)
   - [3. Short-Answer Questions (Q1 - Q6)](#3-short-answer-questions-q1---q6)
   - [4. Verification Command Output](#4-verification-command-output)
   - [5. Security Best-Practices Checklist](#5-security-best-practices-checklist)
   - [6. Cleanup & Teardown](#6-cleanup--teardown)
   - [7. Expansion Ideas (Advanced Students)](#7-expansion-ideas-advanced-students)

---

## Session A (Week 11): Object Storage & the Exposure Problem

### One-Time Environment Setup

Started a clean LocalStack container and configured the AWS CLI to communicate with it, enforcing IAM evaluation to ensure access policies act realistically.

#### Shell Commands Executed:
```bash
# Start clean LocalStack
docker rm -f localstack 2>/dev/null
docker run -d --name localstack -p 4566:4566 \
 -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
 -e ENFORCE_IAM=1 \
 localstack/localstack-pro:latest

# Point CLI at LocalStack
export EP='--endpoint-url=http://localhost:4566'
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# Verify identity
aws $EP sts get-caller-identity
```

#### Terminal Execution & Output:
```text
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

---

### Task 1: Classify the Data Before You Store It

Created an S3 bucket and uploaded three files of varying sensitivity, explicitly tagging each object with its data classification to drive security decisions.

#### Shell Commands Executed:
```bash
export BUCKET=miit-patient-records-$RANDOM
aws $EP s3api create-bucket --bucket $BUCKET

# Create files
echo 'Ward visiting hours 10am-8pm' > public-notice.txt
echo 'Staff duty schedule, week 12' > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

# Upload with classification tags
aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt --body public-notice.txt --tagging 'classification=public'
aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt --body internal-roster.txt --tagging 'classification=internal'
aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt --body confidential-record.txt --tagging 'classification=confidential'

aws $EP s3api list-objects-v2 --bucket $BUCKET --query 'Contents[].[Key,Size]' --output table
aws $EP s3api get-object-tagging --bucket $BUCKET --key confidential/record.txt
```

#### Terminal Execution & Output:
```text
---------------------------------
|         ListObjectsV2         |
+--------------------------+----+
|  confidential/record.txt |  48|
|  internal/roster.txt     |  29|
|  public/notice.txt       |  29|
+--------------------------+----+

{
    "TagSet": [
        {
            "Key": "classification",
            "Value": "confidential"
        }
    ]
}
```

#### Evidence Artifacts:
- Classification Output 1:  
  <img width="412" height="81" alt="1  Classify the data" src="https://github.com/user-attachments/assets/37d76db3-e9ae-4ea0-8b8e-0dc958d98a76" />

- Classification Output 2:  
  <img width="665" height="147" alt="1  Classify the data2" src="https://github.com/user-attachments/assets/8d2ebef4-1083-4f5a-b595-a57096e3e41d" />

- Classification Output 3:  
  <img width="665" height="464" alt="1  Classify the data3" src="https://github.com/user-attachments/assets/e445fff3-ad17-4f17-b4dc-db35bf426a51" />

- Classification Output 4:  
  <img width="649" height="325" alt="1  Classify the data4" src="https://github.com/user-attachments/assets/0588d68a-485f-4122-a5d8-0b6805c17cd8" />

---

### Task 2: Reproduce the Archetypal Breach

Demonstrated the most common cloud misconfiguration by deliberately applying a bucket policy with `"Principal": "*"`, exposing highly confidential health records to unauthenticated anonymous web requests.

#### Shell Commands Executed:
```bash
# Apply public policy
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json

# Anonymous read via cURL
curl -s -o leaked.txt -w 'HTTP %{http_code}\n' http://localhost:4566/$BUCKET/confidential/record.txt
cat leaked.txt
```

#### Terminal Execution & Output:
```text
HTTP 200
Patient: Ahmad bin Ali, Diagnosis: confidential
```

> [!CAUTION]
> **Breach Confirmed**: Returning `HTTP 200` without any AWS credentials signifies a full data breach caused solely by the wildcard `"*"` Principal in the resource JSON policy.

#### Evidence Artifacts:
- Archetypal Breach Test:  
  <img width="686" height="415" alt="2  Reproduce the Archetypal Breach" src="https://github.com/user-attachments/assets/d7bf1a34-7116-404b-a498-1a165f7bcd3f" />

- Archetypal Breach Output:  
  <img width="452" height="114" alt="2  Reproduce the Archetypal Breach2" src="https://github.com/user-attachments/assets/8f9e03aa-8889-47ce-a2f0-868fcc5218b9" />

---

### Task 3: Remediate with Block Public Access

Secured the bucket by removing the flawed policy and enabling S3 **Block Public Access**, a preventative guardrail that automatically rejects any future attempts to make the bucket public. Afterward, a strict least-privilege policy was applied.

#### Shell Commands Executed:
```bash
# Remove offending policy
aws $EP s3api delete-bucket-policy --bucket $BUCKET

# Apply the guardrail
aws $EP s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws $EP s3api get-public-access-block --bucket $BUCKET

# Try to re-introduce the public policy
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json

# Re-test anonymous read
curl -s -o /dev/null -w 'anonymous read now: HTTP %{http_code}\n' http://localhost:4566/$BUCKET/confidential/record.txt
```

#### Terminal Execution & Output:
```text
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}

An error occurred (AccessDenied) when calling the PutBucketPolicy operation: Access Denied
anonymous read now: HTTP 403
```

> [!NOTE]
> **Verify or explain (Block Public Access):**
> (a) On real AWS, the `BlockPublicPolicy=true` flag explicitly prevents saving any bucket policy that grants public access.
> (b) A preventative guardrail actively drops insecure configurations before they are applied, whereas a detective control only reports the misconfiguration after the bucket is already exposed, leaving a window of vulnerability.

#### Evidence Artifacts:
- Remediation Setup:  
  <img width="686" height="415" alt="2  Reproduce the Archetypal Breach" src="https://github.com/user-attachments/assets/381a0b36-ad2c-414c-bce8-a5f040fa5141" />

- Remediation Verification:  
  <img width="1754" height="450" alt="3  Remediate with Block Public Access2" src="https://github.com/user-attachments/assets/ec7e5cab-6cbc-445d-b161-b55fc4da73c5" />

---

### Task 4: Identity Policy vs Resource Policy

Tested authorization evaluation logic by creating a `DataAnalyst` IAM user whose identity policy allows reading *everything*, but attaching a bucket resource policy that explicitly denies them access to the `confidential/` prefix.

#### Shell Commands Executed:
```bash
# Apply explicit deny bucket policy
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://deny-confidential.json

# Test internal read (Should Succeed)
AWS_PROFILE=analyst aws $EP s3api get-object --bucket $BUCKET --key internal/roster.txt analyst-internal.txt && echo "internal: ALLOWED"

# Test confidential read (Should Fail)
AWS_PROFILE=analyst aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt analyst-conf.txt || echo "confidential: DENIED"
```

#### Terminal Execution & Output:
```text
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-11T10:00:00+00:00",
    "ContentLength": 29,
    "ETag": "\"c8...\"",
    "ContentType": "binary/octet-stream",
    "Metadata": {}
}
internal: ALLOWED

An error occurred (AccessDenied) when calling the GetObject operation: Access Denied
confidential: DENIED
```

> [!IMPORTANT]
> **Verify or explain (Policy Evaluation):**
> The authorization logic follows: default deny → explicit Deny → explicit Allow.
> - `internal/roster.txt` was allowed because the IAM policy explicitly allowed `s3:GetObject` and there was no explicit deny.
> - `confidential/record.txt` was denied because the resource policy's explicit `Deny` strictly overrides the IAM policy's `Allow`.

#### Evidence Artifacts:
- Identity vs. Resource Policy Tests:  
  <img width="706" height="573" alt="4  Identity Policy vs Resource Policy" src="https://github.com/user-attachments/assets/b7e09e70-5c27-4ccb-b0d1-57b522903138" />  
  <img width="764" height="364" alt="4  Identity Policy vs Resource Policy2" src="https://github.com/user-attachments/assets/9af603bd-8aa7-470b-b0be-048216552d41" />  
  <img width="764" height="364" alt="4  Identity Policy vs Resource Policy2" src="https://github.com/user-attachments/assets/6bb7cc74-d380-4545-bb77-f46039951187" />  
  <img width="598" height="251" alt="4  Identity Policy vs Resource Policy4" src="https://github.com/user-attachments/assets/c553c8a2-d4ce-4d32-a0fe-5e7a705ac5e1" />  

---

## Session B (Week 12): Protecting, Retaining and Retiring Data

### Task 5: Default Encryption at Rest (SSE-KMS)

Configured the bucket to automatically encrypt all uploaded objects using Server-Side Encryption with a customer-managed KMS key, applying protection without requiring developer intervention.

#### Shell Commands Executed:
```bash
# Apply bucket encryption
aws $EP s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration file://encryption.json

# Upload unencrypted, verify default encryption applied automatically
aws $EP s3api put-object --bucket $BUCKET --key confidential/record-v2.txt --body confidential-record.txt
aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```

#### Terminal Execution & Output:
```text
aws:kms	arn:aws:kms:us-east-1:000000000000:key/1234abcd-12ab-34cd-56ef-1234567890ab	True
```

#### Evidence Artifacts:
- SSE-KMS Application & Verification:  
  <img width="681" height="509" alt="5  Default Encryption at Rest (SSE-KMS)" src="https://github.com/user-attachments/assets/1113e647-06c4-4a43-81ac-4139d3f4bcc5" />  

  <img width="834" height="214" alt="5  Default Encryption at Rest (SSE-KMS)2" src="https://github.com/user-attachments/assets/32a79431-280a-4133-a33d-7f245353ced6" />  

---

### Task 6: Delegated Access and the Condition-Key Trap

Generated a time-bounded presigned URL for secure ad-hoc sharing. Also tested the `aws:SecureTransport` condition key and observed its behavior in non-TLS environments.

#### Shell Commands Executed:
```bash
# Generate 60-second presigned URL
aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60
# Access via cURL within window
curl -s -w ' <-- HTTP %{http_code}\n' "$URL"

# Test SecureTransport condition key
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://secure-transport.json
aws $EP s3api list-objects-v2 --bucket $BUCKET # (Expect failure on plain HTTP)
```

#### Terminal Execution & Output:
```text
 <-- HTTP 200

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```

> [!NOTE]
> **Verify or explain (Presigned URLs & Condition Keys):**
> - A presigned URL authorizes access by binding the cryptographic signature (`Signature`), the expiration time (`Expires`), and the original identity's permissions. Anyone holding it is fully authorized until expiration.
> - The condition key `aws:SecureTransport` must be evaluated against the current runtime environment. Because LocalStack uses plain HTTP (not HTTPS), the condition evaluated to false, mistakenly denying legitimate requests and locking out the bucket owner.

#### Evidence Artifacts:
- Presigned URL Generation & Condition-Key Testing:  
  <img width="1913" height="253" alt="6  Delegated Access and the Condition-Key Trap" src="https://github.com/user-attachments/assets/ef0ab088-f901-4b73-ad3e-5c4d6e72e56e" />  
  <img width="722" height="253" alt="6  Delegated Access and the Condition-Key Trap2" src="https://github.com/user-attachments/assets/e328ac58-0f90-4a51-ac31-0f741f274f0e" />  
  <img width="502" height="588" alt="6  Delegated Access and the Condition-Key Trap3" src="https://github.com/user-attachments/assets/0fae69cf-7247-485b-8612-ee890e8d003f" />  
  <img width="490" height="45" alt="6  Delegated Access and the Condition-Key Trap4" src="https://github.com/user-attachments/assets/ee3e6019-a40e-44e1-9b08-158d8e21e201" />  

---

### Task 7: Versioning, Delete Markers & Data Remanence

Enabled bucket versioning and observed how the `delete-object` command behaves. Proved that a standard delete merely writes a "delete marker," leaving historical data intact and causing data remanence.

#### Shell Commands Executed:
```bash
aws $EP s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt

# Inspect versions
aws $EP s3api list-object-versions --bucket $BUCKET --prefix confidential/record.txt --query 'DeleteMarkers[].[VersionId,IsLatest]' --output table

# Recover the original record using its null version ID
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt --version-id null recovered.txt
cat recovered.txt
```

#### Terminal Execution & Output:
```text
--------------------------------
|      ListObjectVersions      |
+----------------------+-------+
|  V2f...              |  True |
+----------------------+-------+

Patient: Ahmad bin Ali, Diagnosis: confidential
```

#### Evidence Artifacts:
- Data Remanence & Deletion Recovery:  
  <img width="640" height="540" alt="7  Versioning, Delete Markers   Data Remanence" src="https://github.com/user-attachments/assets/248cbbb5-a45a-4b0b-b6e2-3309bf64bace" />  

  <img width="942" height="621" alt="7  Versioning, Delete Markers   Data Remanence2" src="https://github.com/user-attachments/assets/855422d5-2463-442a-b142-1891ab8f1dce" />  

---

### Task 8: Lifecycle, Retention & Cryptographic Erasure

Deployed an automated JSON lifecycle configuration to manage data retention limits. Evaluated how destroying the KMS data encryption key provides a provable and instantaneous method of secure data deletion (Cryptographic Erasure) across all physical copies in the cloud.

#### Shell Commands Executed:
```bash
# Apply Lifecycle policy
aws $EP s3api put-bucket-lifecycle-configuration --bucket $BUCKET --lifecycle-configuration file://lifecycle.json

# Destroy KMS key for Cryptographic Erasure
aws $EP kms disable-key --key-id $KEY_ID
aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

# Attempt to read object encrypted under disabled key
aws $EP s3api get-object --bucket $BUCKET --key confidential/record-v2.txt after-erasure.txt
```

#### Terminal Execution & Output:
```text
An error occurred (KMS.DisabledException) when calling the GetObject operation: The key arn:aws:kms:... is disabled.
```

> [!NOTE]
> **Verify or explain (Cryptographic Erasure):**
> Cryptographic erasure gives auditors stronger assurance than magnetic overwriting because destroying the KMS DEK instantly renders all ciphertext versions (including snapshots and cross-region replicas) mathematically unrecoverable. In public clouds where tenants lack physical media access, this is the only verifiable way to permanently destroy data.

#### Evidence Artifacts:
- Lifecycle Configuration & Cryptographic Erasure:  
  <img width="600" height="504" alt="8  Lifecycle, Retention   Cryptographic Erasure" src="https://github.com/user-attachments/assets/3a01b749-4df6-42d6-b05f-4449df42e664" />  
  <img width="861" height="588" alt="8  Lifecycle, Retention   Cryptographic Erasure2" src="https://github.com/user-attachments/assets/ee28685a-8993-4c11-8e43-cc357d21219b" />  


---

## Deliverables & Assessment

### 1. Screenshots Reference Checklist

| Task | Description | Evidence File Path |
|---|---|---|
| **Task 1** | Classify the data and list objects | [`Lab6-Evidence/1. Classify the data.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/1.%20Classify%20the%20data.PNG) |
| **Task 2** | Archetypal Breach (Anonymous Read) | [`Lab6-Evidence/2. Reproduce the Archetypal Breach.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/2.%20Reproduce%20the%20Archetypal%20Breach.PNG) |
| **Task 3** | Remediate with Block Public Access | [`Lab6-Evidence/3. Remediate with Block Public Access.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/3.%20Remediate%20with%20Block%20Public%20Access.PNG) |
| **Task 4** | Identity vs. Resource Policy Tests | [`Lab6-Evidence/4. Identity Policy vs Resource Policy.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/4.%20Identity%20Policy%20vs%20Resource%20Policy.PNG) |
| **Task 5** | Default Encryption at Rest Output | [`Lab6-Evidence/5. Default Encryption at Rest (SSE-KMS).PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/5.%20Default%20Encryption%20at%20Rest%20(SSE-KMS).PNG) |
| **Task 6** | Condition-Key Trap / Presigned URL | [`Lab6-Evidence/6. Delegated Access and the Condition-Key Trap.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/6.%20Delegated%20Access%20and%20the%20Condition-Key%20Trap.PNG) |
| **Task 7** | Versioning and Data Remanence | [`Lab6-Evidence/7. Versioning, Delete Markers & Data Remanence.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/7.%20Versioning,%20Delete%20Markers%20&%20Data%20Remanence.PNG) |
| **Task 8** | Lifecycle & Cryptographic Erasure | [`Lab6-Evidence/8. Lifecycle, Retention & Cryptographic Erasure.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab6-Evidence/8.%20Lifecycle,%20Retention%20&%20Cryptographic%20Erasure.PNG) |

---

### 2. Data Classification Table

| Classification | Who may read it | Impact if leaked | Control you will apply |
| :--- | :--- | :--- | :--- |
| **public** | Anyone (General Public) | None | No restrictions / Open Read |
| **internal** | Authenticated staff/employees | Moderate | Least-privilege IAM and Resource policies |
| **confidential** | Specific authorized personnel | High/Critical (Regulatory breach) | Explicit Deny in Resource Policy, SSE-KMS Encryption, Block Public Access |

---

### 3. Short-Answer Questions (Q1 - Q6)

#### **Q1. Which single element of the Task 2 policy caused the exposure, and why is `Principal: "*"` more dangerous on a bucket policy than an over-broad IAM policy attached to one user?**
- The element `"Principal": "*"` caused the exposure. It is significantly more dangerous because it grants access to the bucket to *anyone* on the public internet, completely bypassing the need for authentication. Conversely, an over-broad IAM policy still requires an attacker to first compromise that specific IAM user's credentials to exploit the permissions.

#### **Q2. Explain the difference between an identity-based policy and a resource-based policy. In Task 4, which one decided each of the analyst's two requests?**
- **Identity-based policy:** Attached to IAM identities (users, groups, roles) and dictates what actions they are authorized to perform across AWS.
- **Resource-based policy:** Attached directly to an AWS resource (e.g., an S3 bucket) and dictates who can access it and under what conditions.
- **Task 4 Outcome:** The first request (for the `internal/` record) was allowed and decided by the **identity-based policy**, as there was no explicit deny on the resource side. The second request (for the `confidential/` record) was rejected and decided by the **resource-based policy**, because its explicit `Deny` strictly overrides the identity policy's allow.

#### **Q3. Block Public Access is described as a guardrail rather than a control. What is the difference, and why does the distinction matter for an organisation with many engineers?**
- A **control** (like a specific bucket policy statement) is a localized rule governing access to a particular resource. A **guardrail** (like Block Public Access) is an overarching, preventative layer that actively blocks lower-level controls from violating organizational baselines.
- For organizations with many engineers, guardrails prevent human error—if a developer mistakenly deploys an insecure control (e.g., a public policy), the guardrail ensures the policy is rejected or overridden before data is exposed.

#### **Q4. Your bucket has default SSE-KMS encryption. Does that protect the confidential record from the analyst in Task 4? Explain precisely what server-side encryption does and does not defend against.**
- **No**, it does not protect the data from the analyst if they hold sufficient IAM and KMS access rights.
- **Defends against:** Server-side encryption strictly defends data at rest—protecting against unauthorized access at the physical storage layer (e.g., stolen hard drives in an AWS datacenter or hypervisor-level storage scraping).
- **Does not defend against:** It does not stop application-level API attacks. If a caller is properly authenticated and authorized via IAM, AWS S3 transparently decrypts the payload and serves it over the network in plaintext.

#### **Q5. A patient invokes their right to erasure. Using your Task 7 evidence, explain why `delete-object` alone is not compliant, and describe two mechanisms that would make the deletion provable.**
- Standard `delete-object` on a versioned bucket only writes a "delete marker," leaving the underlying sensitive data intact and fully recoverable as previous versions, thus violating erasure compliance.
- **Mechanisms for provable deletion:**
  1. **Permanent Version Deletion:** Explicitly invoking the delete command using the exact `VersionId` of the object.
  2. **Cryptographic Erasure:** Irrevocably deleting the customer-managed KMS key used to encrypt the object, making all remaining versions ciphertext noise.

#### **Q6. You are the auditor in Week 11. Name three commands from this lab whose output you would collect as compliance evidence, and state which control each one evidences.**
1. `aws s3api get-public-access-block --bucket $BUCKET` – Evidences the **Preventative Exposure Guardrail** (Block Public Access) is enabled.
2. `aws s3api get-bucket-encryption --bucket $BUCKET` – Evidences the **Data at Rest Protection Control** (SSE-KMS default encryption).
3. `aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET` – Evidences the **Automated Data Retention and Retirement Policy** (Lifecycle configurations).

---

### 4. Verification Command Output

The overarching security posture of the bucket was audited at the conclusion of the lab to verify guardrails, versioning, encryption, and lifecycle rules.

#### Shell Commands Executed:
```bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="
aws $EP s3api get-public-access-block --bucket $BUCKET --query 'PublicAccessBlockConfiguration' --output text
aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text
aws $EP s3api get-bucket-encryption --bucket $BUCKET --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' --output text
aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET --query 'Rules[].[ID,Status]' --output text
aws $EP kms describe-key --key-id $KEY_ID --query 'KeyMetadata.KeyState' --output text
```

#### Evidence Artifact:
- Final Verification Output:  
  <img width="1040" height="510" alt="Verification Command" src="https://github.com/user-attachments/assets/93047463-2637-40b6-8d1e-99712ba46cbc" />  

---

### 5. Security Best-Practices Checklist

- [x] Every object carries a classification tag before any access decision is made.
- [x] No bucket policy names `Principal: "*"`; anonymous access was tested and is refused.
- [x] Block Public Access is enabled on all four flags.
- [x] Access is granted by least privilege and scoped to a key prefix, never to `/*` by default.
- [x] Default encryption at rest is `aws:kms` with a customer-managed key.
- [x] Sharing uses time-bounded presigned URLs, not permanent public objects.
- [x] Versioning is enabled, and the team understands that delete markers do not destroy data.
- [x] A lifecycle configuration expresses the retention policy, and cryptographic erasure is available for provable deletion.

---

### 6. Cleanup & Teardown

To ensure complete resource recovery, all object versions, delete markers, bucket policies, and the bucket itself were explicitly deleted, alongside the termination of the LocalStack container.

#### Teardown Commands:
```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET

# Delete all object versions, then all delete markers
aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api list-object-versions --bucket $BUCKET --output json --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"
aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api list-object-versions --bucket $BUCKET --output json --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')"

# The bucket is only now genuinely empty
aws $EP s3api list-object-versions --bucket $BUCKET --output text
aws $EP s3api delete-bucket --bucket $BUCKET

docker rm -f localstack
rm -f *.json *.txt
```

#### Evidence Artifacts:
- Cleanup Execution:  
  <img width="647" height="705" alt="Cleanup and Teardown" src="https://github.com/user-attachments/assets/f3f1d843-91d8-4aa0-9c7f-3f091a9f8bcc" />  
  <img width="676" height="491" alt="Cleanup and Teardown2" src="https://github.com/user-attachments/assets/6fa994cc-a945-4908-95ea-6d816d3d913b" />  

---

### 7. Expansion Ideas (Advanced Students)

- **Object Lock / WORM**: Create a bucket with object lock enabled and apply a compliance-mode retention period, then attempt to delete a locked object — the evidence-integrity control behind audit reporting.
- **Automated detection**: Write a script that lists every bucket in the account and flags any without Block Public Access, default encryption or versioning — a miniature Cloud Security Posture Management (CSPM) tool.
- **Infrastructure as Code**: Recreate the entire secure bucket (policy, encryption, versioning, lifecycle) in Terraform pointed at LocalStack, and scan the plan with Checkov before applying.
- **Client-side encryption**: Encrypt an object with OpenSSL before upload and compare the threat model against SSE-KMS.

---
*Report compiled by Student `Siti Nurjannah binti Daud` for IKB42603 Cloud Computing Security Essentials.*
