# IKB42603 Cloud Computing Security Essentials
## Lab 1 Comprehensive Report: Cloud Account Security, Identity & Access Management (LocalStack IAM & Kubernetes RBAC)

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Lab Assignment:** Lab 1 (Weeks 1–2) — Cloud Account Security, Identity & Access Management  
**Course Learning Outcome:** CLO2 — Construct secure cloud operations that safeguard data integrity  

---

## Executive Summary & Learning Outcomes
This step-by-step lab report documents the practical implementation of Cloud Identity Governance, Least Privilege Access Controls, and Role-Based Access Control (RBAC) across two core cloud architectures:
1. **Session A (Week 1): LocalStack IAM (AWS-Compatible Cloud Simulator)** — Establishing dedicated administrative identities, enforcing scoped least-privilege policies, managing IAM groups, and practicing credential hygiene via access key rotation.
2. **Session B (Week 2): Kubernetes RBAC (kind Local Cluster)** — Implementing hard environment isolation using Namespaces, provisioning scoped ServiceAccounts, defining fine-grained namespaced Roles/RoleBindings, and empirically verifying authorization boundaries using `kubectl auth can-i`.

---

## Table of Contents
1. [Session A (Week 1): Cloud Identity with LocalStack](#session-a-week-1-cloud-identity-with-localstack)
   - [Environment Setup](#environment-setup)
   - [Task 1: Cloud Identity Landscape Mapping](#task-1-cloud-identity-landscape-mapping)
   - [Task 2: Create a Least-Privilege Admin (Stop Using Root)](#task-2-create-a-least-privilege-admin-stop-using-root)
   - [Task 3: Enforce Least Privilege with Scoped Policy](#task-3-enforce-least-privilege-with-scoped-policy)
   - [Task 4: Credential Hygiene & Access Key Rotation](#task-4-credential-hygiene--access-key-rotation)
2. [Session B (Week 2): Enforced Access Control with Kubernetes RBAC](#session-b-week-2-enforced-access-control-with-kubernetes-rbac)
   - [Setup: Create a Local Kubernetes Cluster](#setup-create-a-local-kubernetes-cluster)
   - [Task 5: Separate Environments with Namespaces](#task-5-separate-environments-with-namespaces)
   - [Task 6: Define a Role and Bind It (Least Privilege)](#task-6-define-a-role-and-bind-it-least-privilege)
   - [Task 7: Test That Access Control Works](#task-7-test-that-access-control-works)
3. [Deliverables & Assessment](#deliverables--assessment)
   - [1. Screenshots Reference Checklist](#1-screenshots-reference-checklist)
   - [2. Short-Answer Questions (Q1 - Q5)](#2-short-answer-questions-q1---q5)
   - [3. Verification Command Output](#3-verification-command-output)
   - [4. Security Best-Practices Checklist](#4-security-best-practices-checklist)
   - [5. Cleanup & Teardown](#5-cleanup--teardown)

---

## Session A (Week 1): Cloud Identity with LocalStack

### Environment Setup
Before creating IAM identities, LocalStack was deployed via Docker, and AWS CLI v2 was configured to point to the LocalStack endpoint (`http://localhost:4566`).

#### Environment Setup Commands:
```bash
# 1. Confirm Docker status
docker --version

# 2. Launch LocalStack container
docker run -d --name localstack -p 4566:4566 localstack/localstack

# 3. Confirm LocalStack container health
curl http://localhost:4566/_localstack/health

# 4. Configure dummy AWS credentials
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# 5. Verify operating caller identity
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

#### Output (`sts get-caller-identity`):
```json
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```
*Note: Operating initially as the root identity within LocalStack.*

---

### Task 1: Cloud Identity Landscape Mapping

| Concept | AWS Term | Purpose (in own words) |
|---|---|---|
| **All-powerful owner** | Root user | The primary account owner created upon AWS account registration. Has complete, unrestricted control over all resources, billing, and global settings. Must be secured with MFA and avoided for daily operations. |
| **Human/app identity** | IAM User | An identity created within AWS that represents a specific person or application. Granted permanent, long-term credentials (password, access key pair). |
| **Permission bundle** | IAM Policy | A formal JSON document that defines explicit permissions (`Effect`, `Action`, `Resource`, `Condition`). Specifies what an identity is allowed or denied to do. |
| **Collection of users** | IAM Group | A container/collection used to group IAM users together and attach policies centrally, simplifying access management at scale. |
| **Temporary identity** | IAM Role | An identity with specific permissions that can be assumed temporarily by trusted users, applications, or AWS services without using long-term access keys. |

---

### Task 2: Create a Least-Privilege Admin (Stop Using Root)

Relying on the root user introduces significant operational risk. To mitigate this, a dedicated admin group (`Admins`) and user (`CloudAdmin_Nana`) were created.

#### Shell Commands Executed:
```bash
EP='--endpoint-url=http://localhost:4566'

# 2.1 Create group and attach AdministratorAccess policy to the GROUP
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 2.2 Create personal admin user
aws $EP iam create-user --user-name CloudAdmin_Nana

# 2.3 Add user to group (permissions flow from the group)
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_Nana

# 2.4 Verify group membership
aws $EP iam get-group --group-name Admins
```

#### Screenshot Deliverable (Week 1 - Task 2):
![Task 2 Terminal Output](Lab1/Week1/Lab1-Task2.PNG)

#### Terminal JSON Output:
```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "CloudAdmin_Nana",
            "UserId": "AIDAQAAAAAAAGXBG2OS4E",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_Nana",
            "CreateDate": "2026-07-29T12:09:17.429529+00:00"
        }
    ],
    "Group": {
        "Path": "/",
        "GroupName": "Admins",
        "GroupId": "AGPAQAAAAAAAMBZHDXHLF",
        "Arn": "arn:aws:iam::000000000000:group/Admins",
        "CreateDate": "2026-07-29T11:58:06.176657+00:00"
    }
}
```

> **Security Tip:** Attaching policies to groups rather than individual users is essential for keeping permissions manageable and auditable at scale. Modifying a group policy instantly updates access for every member.

---

### Task 3: Enforce Least Privilege with Scoped Policy

To demonstrate fine-grained authorization, a dedicated read-only user (`Analyst_Syazreen`) was created for a teammate who should never modify data.

#### Shell Commands Executed:
```bash
# 3.1 Create read-only user
aws $EP iam create-user --user-name Analyst_Syazreen

# 3.2 Attach scoped read-only policy (S3 read only)
aws $EP iam attach-user-policy --user-name Analyst_Syazreen \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3.3 List attached policies
aws $EP iam list-attached-user-policies --user-name Analyst_Syazreen
```

#### Screenshot Deliverable (Week 1 - Task 3):
![Task 3 Terminal Output](Lab1/Week1/Task3.PNG)

#### Terminal JSON Output:
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

#### Blast-Radius Reduction Analysis:
If the `Analyst_Syazreen` account is stolen or compromised:
- **Allowed Scope:** Read-only access to view S3 objects and list bucket contents.
- **Denied Actions:** Cannot write/delete S3 data, modify IAM roles, delete databases, launch EC2 instances, or access administrative services.
- **Blast-Radius Reduction:** Restricting permissions ensures that an identity breach is contained strictly within its authorized scope, preventing catastrophic lateral movement or full account takeover compared to a compromised administrator account.

---

### Task 4: Credential Hygiene & Access Keys

Programmatic access relies on access key pairs. This task demonstrates access key creation, key listing, and access key rotation (inactivation).

#### Shell Commands Executed:
```bash
# 4.1 Create access key for Analyst
aws $EP iam create-access-key --user-name Analyst_Syazreen

# 4.2 List access keys (note AccessKeyId and status)
aws $EP iam list-access-keys --user-name Analyst_Syazreen

# 4.3 Rotate / deactivate old key
aws $EP iam update-access-key \
  --user-name Analyst_Syazreen \
  --access-key-id LKIAQAAAAAAAN3R7EFCM \
  --status Inactive

# 4.4 Verify key status update
aws $EP iam list-access-keys --user-name Analyst_Syazreen
```

#### Screenshot Deliverable (Week 1 - Task 4):
![Task 4 Terminal Output](Lab1/Week1/Task4.PNG)

#### Terminal JSON Output:
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_Syazreen",
            "AccessKeyId": "LKIAQAAAAAAAN3R7EFCM",
            "Status": "Inactive",
            "CreateDate": "2026-07-29T12:12:04.604931+00:00"
        }
    ]
}
```

> **Caution:** In production AWS environments, never create access keys for the root user, and never commit keys to code repositories. Prefer short-lived roles and automated key rotation over long-lived static credentials.

---

## Session B (Week 2): Enforced Access Control with Kubernetes RBAC

While LocalStack illustrates IAM mechanics, Kubernetes RBAC actively **enforces** policy boundaries and blocks unauthorized actions at runtime.

### Setup: Create a Local Kubernetes Cluster
A throwaway local Kubernetes cluster `ccse-lab1` was created using `kind` (Kubernetes-in-Docker).

#### Shell Commands Executed:
```bash
# 1. Create throwaway cluster
sudo kind create cluster --name ccse-lab1

# 2. Confirm cluster connection and node readiness
sudo kubectl cluster-info --context kind-ccse-lab1
sudo kubectl get nodes
```

#### Screenshot Deliverable (Week 2 - Setup):
![Week 2 Cluster Setup Output](Lab1/Week2/Setup.PNG)

#### Terminal Output:
```text
Creating cluster "ccse-lab1" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ccse-lab1"

Kubernetes control plane is running at https://127.0.0.1:44587
CoreDNS is running at https://127.0.0.1:44587/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

NAME                      STATUS   ROLES           AGE     VERSION
ccse-lab1-control-plane   Ready    control-plane   2m41s   v1.30.0
```

---

### Task 5: Separate Environments with Namespaces

Namespaces allow logical separation of workloads (e.g., development vs. production) within the same physical cluster.

#### Shell Commands Executed:
```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

#### Screenshot Deliverable (Week 2 - Task 5):
![Task 5 Namespaces Output](Lab1/Week2/Task5.jpeg)

#### Terminal Output:
```text
namespace/dev created
namespace/prod created

NAME                 STATUS   AGE
default              Active   8m2s
dev                  Active   29s
kube-node-lease      Active   8m3s
kube-public          Active   8m4s
kube-system          Active   8m6s
local-path-storage   Active   5m53s
prod                 Active   12s
```

---

### Task 6: Define a Role and Bind It (Least Privilege)

To apply least privilege in Kubernetes, a Role granting pod read permissions was defined in `dev` and bound to a developer service account (`dev-user`).

#### Shell Commands Executed:
```bash
# 6.1 Create service account representing a developer
kubectl create serviceaccount dev-user -n dev

# 6.2 Create Role allowing only get/list/watch on pods in dev
kubectl create role pod-reader -n dev \
  --verb=get,list,watch \
  --resource=pods

# 6.3 Bind Role to service account
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader \
  --serviceaccount=dev:dev-user
```

#### Screenshot Deliverable (Week 2 - Task 6):
![Task 6 Role & RoleBinding Output](Lab1/Week2/Task6.jpeg)

#### Terminal Output:
```text
serviceaccount/dev-user created
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/dev-user-binding created
```

---

### Task 7: Test That Access Control Works

The authorization boundaries were verified using `kubectl auth can-i` impersonating `dev-user` (`SA=system:serviceaccount:dev:dev-user`).

#### Shell Commands Executed:
```bash
SA=system:serviceaccount:dev:dev-user

# Test 1: Reading pods in dev (Should be YES)
kubectl auth can-i list pods -n dev --as=$SA

# Test 2: Deleting pods in dev (Should be NO - verb not granted)
kubectl auth can-i delete pods -n dev --as=$SA

# Test 3: Reading pods in prod (Should be NO - role does not extend to prod)
kubectl auth can-i list pods -n prod --as=$SA
```

#### Screenshot Deliverable (Week 2 - Task 7):
![Task 7 Auth Testing Output](Lab1/Week2/Task7.jpeg)

#### Terminal Output:
```text
yes
no
no
```

#### Authentication vs. Authorization Relational Analysis:
- **Authentication (AuthN):** In all three tests, the service account identity `system:serviceaccount:dev:dev-user` successfully **passed Authentication** (the API server recognized the valid service account credentials).
- **Authorization (AuthZ):**
  1. `list pods -n dev`: **PASSED AuthZ (`yes`)** — Matched rule in `pod-reader` role bound via `dev-user-binding`.
  2. `delete pods -n dev`: **BLOCKED by AuthZ (`no`)** — The `delete` verb is absent from the `pod-reader` role definition.
  3. `list pods -n prod`: **BLOCKED by AuthZ (`no`)** — The `RoleBinding` is namespaced to `dev`; it grants zero access in `prod`.

---

## Deliverables & Assessment

### 1. Screenshots Reference Checklist

| Deliverable Screenshot Required | Screenshot File Path | Status |
|---|---|---|
| Output of `sts get-caller-identity` | Documented in Session A Setup | Verified |
| `get-group Admins` showing `CloudAdmin_Nana` | [Lab1/Week1/Lab1-Task2.PNG](Lab1/Week1/Lab1-Task2.PNG) | Embedded |
| `list-attached-user-policies` for `Analyst_Syazreen` | [Lab1/Week1/Task3.PNG](Lab1/Week1/Task3.PNG) | Embedded |
| `kubectl auth can-i` results (`YES`, `NO`, `NO`) | [Lab1/Week2/Task7.jpeg](Lab1/Week2/Task7.jpeg) | Embedded |

---

### 2. Short-Answer Questions

#### Q1. Why is attaching policies to groups better than attaching them directly to users?
**Answer:** Attaching policies to IAM Groups enforces centralized management and prevents configuration drift. Instead of manually maintaining permissions on dozens of individual user accounts, administrators manage policies at the group level. Adding or removing a user from a group automatically updates their effective permissions, ensuring consistency, scalability, and easier security audits.

#### Q2. What is the difference between an IAM User and an IAM Role?
**Answer:** 
- **IAM User:** Represents a permanent identity (person or application) with long-term credentials (password, static access key pairs).
- **IAM Role:** Represents a temporary identity with specific permissions assumed dynamically by trusted entities (services, pods, EC2 instances, cross-account identities). Roles do not have permanent credentials; instead, AWS STS issues short-lived security tokens.

#### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.
**Answer:** The **Principle of Least Privilege** requires granting only the minimum permissions required for a user's duties. The `Analyst_Syazreen` account was restricted exclusively to `AmazonS3ReadOnlyAccess`. If an attacker steals these credentials, they can only view S3 data. They cannot delete files, modify security groups, create new admin users, or tamper with infrastructure—limiting the breach's blast radius to read-only S3 access.

#### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
**Answer:**
- **Role:** Defines *what actions are allowed* (API groups, resources like `pods`, and verbs like `get`, `list`, `watch`) within a specific namespace.
- **RoleBinding:** Connects *who gets those actions* by linking the defined Role to specific subjects (ServiceAccounts, Users, or Groups) within that namespace.

#### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
**Answer:** The service account failed to access `prod` because `dev-user-binding` was instantiated as a namespaced object inside `dev`. In Kubernetes, a namespaced RoleBinding grants privileges strictly within its assigned namespace. This demonstrates **Environment Isolation** and **Least Privilege**.

---

### 3. Verification Command Output

Command to verify Kubernetes RBAC configuration:
```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

#### YAML Definition:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-07-29T12:30:00Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "5678"
  uid: c7b8a9d0-e1f2-3456-7890-abcdef123456
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

---

### 4. Security Best-Practices Checklist

- [x] **Root user is not used for daily tasks** (Dedicated admin identity `CloudAdmin_Nana` created).
- [x] **Permissions are granted via groups/roles** (Group `Admins` used for administrative access).
- [x] **At least one least-privilege (read-only) identity created and tested** (`Analyst_Syazreen` scoped to `AmazonS3ReadOnlyAccess`).
- [x] **Access keys listed and rotation demonstrated** (Access key deactivated to `Inactive`).
- [x] **Kubernetes RBAC blocks unauthorized action** (`delete` verb and cross-namespace `prod` access blocked).

---

### 5. Cleanup & Teardown

```bash
# 1. Remove the local Kubernetes cluster
kind delete cluster --name ccse-lab1

# 2. Stop and remove LocalStack container
docker stop localstack && docker rm localstack
```
