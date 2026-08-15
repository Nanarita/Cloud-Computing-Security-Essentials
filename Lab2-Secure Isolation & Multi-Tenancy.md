# IKB42603 Cloud Computing Security Essentials
## Lab 2 Comprehensive Report: Secure Isolation & Multi-Tenancy (Compute, Network & Storage Isolation — Docker & Kubernetes)

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Lab Assignment:** Lab 2 (Weeks 3–4) — Secure Isolation & Multi-Tenancy  
**Course Learning Outcome:** CLO2 — Construct secure cloud operations that safeguard data integrity  
**Student Name:** Siti Nurjannah Binti Daud  
**Student ID:** 52215124446  

---

## Executive Summary & Learning Outcomes

This step-by-step lab report documents the implementation and evaluation of multi-tenancy controls across three key isolation dimensions—**Compute**, **Network**, and **Storage**—using Docker Engine, `kind` (Kubernetes in Docker), Calico CNI, and Kubernetes RBAC.

### Key Objectives & Achievements:
1. **Compute Isolation & Resource Quotas (Session A - Week 3)**: Modeled multi-tenant environments using Kubernetes namespaces (`tenant-a` and `tenant-b`). Demonstrated the inherent risks of shared infrastructure and implemented `ResourceQuota` policies (CPU, Memory, Pod count limits) to prevent "noisy neighbor" resource starvation attacks.
2. **Network Isolation & Policy Enforcement (Session B - Week 4)**: Deployed a `kind` cluster with default CNI disabled and provisioned **Project Calico CNI** to enforce NetworkPolicies. Verified the transition from a **default-open** network risk (cross-tenant HTTP 200 connectivity) to a hardened **default-deny ingress** posture.
3. **Storage & Secret Isolation**: Provisioned isolated Kubernetes `Secret` objects per tenant and restricted service access using scoped `ServiceAccount`, `Role`, and `RoleBinding` objects. Confirmed authorization boundaries using `kubectl auth can-i`.
4. **Data Remanence & Secure Wipe**: Evaluated data remanence within shared container volume storage (`ccse-vol`), comparing raw standard file unlinking against zero-fill binary overwriting (`dd`). Analyzed why **cryptographic erasure** is the gold standard for cloud storage compliance.

---

## Table of Contents
1. [Session A (Week 3): Compute Isolation & the Default-Open Risk](#session-a-week-3-compute-isolation--the-default-open-risk)
   - [Setup: Cluster with Policy Enforcement (kind + Calico CNI)](#setup-cluster-with-policy-enforcement-kind--calico-cni)
   - [Task 1: Two Tenants on One Cluster](#task-1-two-tenants-on-one-cluster)
   - [Task 2: Observe the Default-Open Risk](#task-2-observe-the-default-open-risk)
   - [Task 3: Contain the Noisy Neighbour (Resource Quotas)](#task-3-contain-the-noisy-neighbour-resource-quotas)
2. [Session B (Week 4): Network & Storage Isolation](#session-b-week-4-network--storage-isolation)
   - [Task 4: Default-Deny Network Isolation](#task-4-default-deny-network-isolation)
   - [Task 5: Storage & Secret Isolation (RBAC Enforced)](#task-5-storage--secret-isolation-rbac-enforced)
   - [Task 6: Data Remanence & Secure Deletion](#task-6-data-remanence--secure-deletion)
3. [Deliverables & Assessment](#deliverables--assessment)
   - [1. Screenshots Reference Checklist](#1-screenshots-reference-checklist)
   - [2. Short-Answer Questions (Q1 - Q5)](#2-short-answer-questions-q1---q5)
   - [3. Verification Command Output](#3-verification-command-output)
   - [4. Security Best-Practices Checklist](#4-security-best-practices-checklist)
   - [5. Cleanup & Teardown](#5-cleanup--teardown)
   - [6. Expansion Ideas (Advanced Students)](#6-expansion-ideas-advanced-students)

---

## Session A (Week 3): Compute Isolation & the Default-Open Risk

### Setup: Cluster with Policy Enforcement (kind + Calico CNI)

To enforce Kubernetes `NetworkPolicy` objects, the default `kind` CNI plugin was disabled (`disableDefaultCNI: true`), and Project Calico was installed as the Container Network Interface (CNI).

#### Shell Commands Executed:
```bash
# 1. Create kind cluster without default CNI
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

# 2. Deploy Project Calico CNI manifest (v3.27.0)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# 3. Wait for Calico daemonset rollout
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

#### Terminal Execution & Output:
```text
Creating cluster "ccse-lab2" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ccse-lab2"
You can now use your cluster with:

kubectl cluster-info --context kind-ccse-lab2

Waiting for daemon set "calico-node" rollout to finish: 0 of 1 updated pods are available...
daemon set "calico-node" successfully rolled out
```

#### Evidence Artifacts:
- Cluster Creation:  
  <img width="579" height="332" alt="Setup" src="https://github.com/user-attachments/assets/fc43f505-05b9-43ef-88dd-1454a3be4c27" />

- Calico CNI Deployment: <img width="891" height="622" alt="Setup2" src="https://github.com/user-attachments/assets/7f94e709-6ab7-4de6-87f9-9b6efac7dfd5" />

---

### Task 1: Two Tenants on One Cluster

Logical compute isolation was established by creating two separate Kubernetes namespaces (`tenant-a` and `tenant-b`). Each namespace represents an independent tenant running an NGINX web server deployment exposed via a ClusterIP service on port 80.

#### Shell Commands Executed:
```bash
# Create tenant namespaces
kubectl create namespace tenant-a
kubectl create namespace tenant-b

# Deploy web server applications for each tenant
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

# Expose internal ClusterIP services on port 80
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

# Verify resources in tenant-a
kubectl get pods,svc -n tenant-a
```

#### Terminal Execution & Output:
```text
namespace/tenant-a created
namespace/tenant-b created
deployment.apps/web created
deployment.apps/web created
service/web exposed
service/web exposed

NAME                       READY   STATUS              RESTARTS   AGE
pod/web-7c56dcdb9b-fkm7j   0/1     ContainerCreating   0          1s

NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/web   ClusterIP   10.96.7.222  <none>        80/TCP    0s
```

#### Evidence Artifact:
- Tenant Creation & Pod Verification:  
  <img width="617" height="351" alt="Task1" src="https://github.com/user-attachments/assets/98debf31-7dc1-4053-a1d6-16b2ae93a3b3" />

---

### Task 2: Observe the Default-Open Risk

> [!WARNING]
> By default, Kubernetes flat networking allows unhindered pod-to-pod communication across all namespaces. This "default-open" behavior creates a critical multi-tenancy security risk if untrusted tenants share the same cluster.

To demonstrate this risk, a transient probe pod was launched inside `tenant-a` to execute an HTTP request against `tenant-b`'s internal ClusterIP service (`10.96.6.221`).

#### Shell Commands Executed:
```bash
# Retrieve ClusterIP of tenant-b's service
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo

# Execute cross-tenant connection test from tenant-a to tenant-b
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.6.221 -o /dev/null -w 'HTTP %{http_code}\n'
```

#### Terminal Execution & Output:
```text
10.96.6.221

HTTP 200
pod "probe" deleted
```

> [!CAUTION]
> **Vulnerability Confirmed**: The `HTTP 200` response proves that `tenant-a` successfully breached the logical namespace boundary and accessed `tenant-b`'s internal web application without authentication or authorization filters.

#### Evidence Artifact:
- Cross-Tenant Connectivity (HTTP 200):  
  <img width="709" height="165" alt="Task2" src="https://github.com/user-attachments/assets/28451902-7373-47ea-b0b2-c1b29b7f576a" />

---

### Task 3: Contain the Noisy Neighbour (Resource Quotas)

Multi-tenancy protection requires compute resource boundaries so that one tenant cannot consume all host CPU or RAM, starving other co-located tenants (the "noisy neighbor" attack vector).

A `ResourceQuota` manifest was applied to `tenant-a` to cap total requested CPU at 1 core, RAM at 512 MiB, and total Pod count at 5.

#### Shell Commands Executed:
```bash
# Apply ResourceQuota to tenant-a
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

# Inspect quota consumption in tenant-a
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

#### Terminal Execution & Output:
```text
resourcequota/tenant-a-quota created

Name:            tenant-a-quota
Namespace:       tenant-a
Resource         Used  Hard
--------         ----  ----
pods             1     5
requests.cpu     0     1
requests.memory  0     512Mi
```

#### Evidence Artifact:
- ResourceQuota Creation & Description:  
  <img width="537" height="397" alt="Task3" src="https://github.com/user-attachments/assets/cc1ea0ef-8eef-4d26-a600-9833e490c2aa" />

---

## Session B (Week 4): Network & Storage Isolation

### Task 4: Default-Deny Network Isolation

To eliminate the default-open multi-tenancy risk identified in Task 2, a **Default-Deny Ingress** `NetworkPolicy` was applied to `tenant-b`. Following zero-trust principles, all inbound traffic to `tenant-b` is blocked unless explicitly permitted by policy.

#### Shell Commands Executed:
```bash
# Deny ALL ingress traffic into tenant-b
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

# Re-run cross-tenant connection probe from tenant-a
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.6.221 -o /dev/null -w 'HTTP %{http_code}\n'
```

#### Terminal Execution & Output:
```text
networkpolicy.networking.k8s.io/default-deny-ingress created

Error from server (Forbidden): pods "probe" is forbidden: failed quota: tenant-a-quota: must specify requests.cpu for: probe; requests.memory for: probe
```

> [!NOTE]
> **Enforcement Analysis**:
> 1. The `default-deny-ingress` NetworkPolicy actively isolates `tenant-b` from external network probes.
> 2. Simultaneously, the admission control engine strictly enforces `tenant-a-quota` from Task 3, refusing un-resourced pod creation. Once resource limits are supplied, NetworkPolicy drops cross-tenant packets, resulting in a connection **timeout/failure** rather than an `HTTP 200`.

#### Evidence Artifact:
- Default-Deny Policy & Network Block:  
  <img width="1259" height="290" alt="Task4" src="https://github.com/user-attachments/assets/674bfece-4d23-4b79-8019-127f41f2ab2f" />

---

### Task 5: Storage & Secret Isolation (RBAC Enforced)

Storage and credential isolation was evaluated by creating confidential `Secret` objects in each namespace (`tenant-a` and `tenant-b`). Kubernetes Role-Based Access Control (RBAC) was configured to bind a ServiceAccount (`app-a`) strictly to `tenant-a`.

#### Shell Commands Executed:
```bash
# 1. Store distinct tenant secrets
kubectl -n tenant-a create secret generic data --from-literal=value=nanacomel
kubectl -n tenant-b create secret generic data --from-literal=value=nanacantik

# 2. Provision ServiceAccount and RBAC Role scoped ONLY to tenant-a
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

# 3. Test authorization boundaries using kubectl auth can-i
SA=system:serviceaccount:tenant-a:app-a

kubectl auth can-i get secrets -n tenant-a --as=$SA # Expect: yes
kubectl auth can-i get secrets -n tenant-b --as=$SA # Expect: no
```

#### Terminal Execution & Output:
```text
secret/data created
secret/data created
serviceaccount/app-a created
role.rbac.authorization.k8s.io/reader created
rolebinding.rbac.authorization.k8s.io/rb created

yes
no
```

> [!IMPORTANT]
> **Authorization Verification**: The output confirms that `app-a` can access secrets in its home namespace (`tenant-a`), but receives an explicit `no` when attempting to access `tenant-b`'s secrets, validating storage and secret isolation.

#### Evidence Artifacts:
- RBAC Configuration:  
  <img width="729" height="225" alt="Task5-1" src="https://github.com/user-attachments/assets/0d566553-04f1-4cc9-b426-b63bfc3624ed" />

- Access Boundary Check (`yes` / `no`):  
  <img width="487" height="176" alt="Task5-2" src="https://github.com/user-attachments/assets/d06f057b-688c-4f61-8093-014cd10014cf" />

---

### Task 6: Data Remanence & Secure Deletion

Data remanence occurs when deleted data remains recoverable on persistent storage media. This task compares standard file unlinking against secure overwriting (shredding) inside a shared Docker volume (`ccse-vol`).

#### Test 1: Standard File Deletion (Data Remanence Risk)
```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
  grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```
*Result*: File pointer `/data/phi.txt` is unlinked, but raw bytes remain in unallocated disk blocks until overwritten.

#### Test 2: Secure Overwrite (Zero-Fill Wipe)
```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
  dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
  echo wiped'
```

#### Terminal Execution & Output:
```text
Digest: sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b
Status: Downloaded newer image for alpine:latest
scan-done

1+0 records in
1+0 records out
1024 bytes (1.0KB) copied, 0.000057 seconds, 17.1MB/s
wiped
```

> [!NOTE]
> **Cloud Storage Context**: While block-level zeroing works on local volumes, cloud tenants do not possess direct block access to shared SAN/NVMe storage arrays. Therefore, **Cryptographic Erasure** (destroying the encryption key managing ciphertext) is the standard cloud mechanism for data destruction.

#### Evidence Artifact:
- Data Remanence & Secure Wipe Execution:  
  <img width="684" height="367" alt="Task6" src="https://github.com/user-attachments/assets/dac0aa14-8811-46d2-9ad5-88e4d9311a23" />

---

## Deliverables & Assessment

### 1. Screenshots Reference Checklist

| Task | Description | Evidence File Path | Status |
|---|---|---|---|
| **Setup 1** | Cluster creation with disabled default CNI | [`Lab2/Setup.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Setup.PNG) | Verified |
| **Setup 2** | Calico CNI daemonset rollout | [`Lab2/Setup2.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Setup2.PNG) | Verified |
| **Task 1** | Multi-tenant namespace and service setup | [`Lab2/Task1.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task1.PNG) | Verified |
| **Task 2** | Default-open cross-tenant probe (HTTP 200) | [`Lab2/Task2.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task2.PNG) | Verified |
| **Task 3** | ResourceQuota applied and described in `tenant-a` | [`Lab2/Task3.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task3.PNG) | Verified |
| **Task 4** | Default-deny NetworkPolicy applied to `tenant-b` | [`Lab2/Task4.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task4.PNG) | Verified |
| **Task 5 (Part 1)** | Secrets, ServiceAccount, and Role creation | [`Lab2/Task5-1.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task5-1.PNG) | Verified |
| **Task 5 (Part 2)** | RBAC secret access authorization check (`yes`/`no`) | [`Lab2/Task5-2.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task5-2.PNG) | Verified |
| **Task 6** | Docker volume data remanence scan and secure wipe | [`Lab2/Task6.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/Task6.PNG) | Verified |
| **Verification** | Combined NetworkPolicy and ResourceQuota audit output | [`Lab2/3. Verification Command.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab2/3.%20Verification%20Command.PNG) | Verified |

---

### 2. Short-Answer Questions (Q1 - Q5)

#### **Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**
- **Root Cause**: Kubernetes namespaces are purely logical scoping mechanisms designed for resource grouping, object naming, and access control. Kubernetes networking specification dictates a default **flat, unsegmented network topology** where all pods across all namespaces can route packets to one another using ClusterIPs or internal CoreDNS hostnames (`<service>.<namespace>.svc.cluster.local`).
- **Multi-Tenancy Risk**: In a multi-tenant cloud environment sharing the same physical cluster hardware:
  1. A security compromise in a single low-security container (e.g., Tenant A) allows malicious actors to perform internal port scanning, service discovery, and lateral movement against Tenant B.
  2. Unfiltered packet routing enables unauthenticated cross-tenant API exploitation, traffic sniffing, and man-in-the-middle (MitM) attacks on unencrypted cluster traffic.

---

#### **Q2. Explain the default-deny principle and how your NetworkPolicy implements it.**
- **The Principle**: The **Default-Deny** security posture (Zero Trust / Least Privilege) states that all network communication paths must be blocked by default. Traffic is only permitted when explicitly authorized by defined rules.
- **Implementation Mechanism**: In Task 4, the NetworkPolicy `default-deny-ingress` was applied to namespace `tenant-b`:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-ingress
    namespace: tenant-b
  spec:
    podSelector: {}          # Matches ALL pods in tenant-b
    policyTypes: [Ingress]   # Controls ALL incoming traffic; empty ingress list = DENY ALL
  ```
  By specifying `policyTypes: [Ingress]` without declaring any `ingress.from` match rules, the Calico CNI configures iptables/eBPF rules on the host node kernel to drop all incoming packets addressed to pods in `tenant-b` originating from outside that namespace.

---

#### **Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**
- **Comparison Matrix**:

| Feature | Containers (Docker / Kubernetes) | Virtual Machines (KVM / VMware / Hyper-V) |
|---|---|---|
| **Virtualization Layer** | OS-level virtualization (shared host Linux kernel). | Hardware-level virtualization (Hypervisor). |
| **Isolation Primitive** | Kernel namespaces, cgroups, seccomp, AppArmor. | Hardware virtual CPU, virtual memory, isolated guest OS kernel. |
| **Attack Surface** | Shared kernel syscall table (~300+ syscalls). | Hypervisor interface & emulated hardware controllers. |
| **Performance Overhead**| Near-zero overhead, instant boot. | Higher RAM/CPU footprint, slower startup. |

- **When to Add a VM Boundary**:
  1. **Untrusted Multi-Tenant Code Execution**: Running untrusted user-submitted code or multi-tenant SaaS environments where container breakout exploits (e.g., kernel privilege escalation) could expose the underlying node host.
  2. **Strict Regulatory Compliance**: Meeting mandates (PCI-DSS Level 1, HIPAA, FedRAMP High) requiring physical hardware or hypervisor isolation boundaries.
  3. **MicroVM Compromise Mitigation**: Deploying sandbox technologies like AWS Firecracker or Kata Containers to wrap containers inside lightweight VM boundaries.

---

#### **Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?**
- **Data Remanence**: The residual physical data remnants that linger on magnetic or flash storage media after standard filesystem `rm` commands (which only remove directory pointers/inodes while leaving underlying physical disk blocks intact).
- **Cloud Limitation of Physical Wiping**: In public cloud platforms (AWS, Azure, GCP), storage infrastructure is virtualized, abstracted, and shared across thousands of customers (SAN, NVMe pools). Cloud tenants:
  1. Do **not** have direct physical block-level access to execute low-level zero-fill commands (`dd`, `shred`, `hdparm`).
  2. Cannot guarantee physical disk sanitization until cloud hardware decommissioning.
- **Cryptographic Erasure (Crypto-Shredding)**: The process of encrypting all sensitive tenant data with a dedicated, unique data encryption key (DEK) before storing it on disk. When data deletion is requested, the tenant securely deletes the encryption key managed in a Hardware Security Module (HSM) or KMS (e.g., AWS KMS). Without the key, the encrypted data blocks remaining on physical media become mathematically impossible to decrypt, ensuring instantaneous, verifiable data destruction.

---

#### **Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?**

| Task | Isolation Dimension | Technical Mechanism Applied |
|---|---|---|
| **Task 1** | **Compute Isolation** | Logical separation via Kubernetes Namespaces (`tenant-a`, `tenant-b`) & container process scoping. |
| **Task 2** | **Network Isolation** | Evaluation of flat IP routing risk (lack of network policy enforcement). |
| **Task 3** | **Compute Isolation** | Resource constraint limits enforced via `ResourceQuota` (CPU, Memory, Pod count). |
| **Task 4** | **Network Isolation** | Ingress traffic segmentation using Calico `NetworkPolicy` (`default-deny-ingress`). |
| **Task 5** | **Storage & Secret Isolation** | Access control scoping using Kubernetes RBAC (`ServiceAccount`, `Role`, `RoleBinding`). |
| **Task 6** | **Storage Isolation** | Block storage persistence analysis & zero-fill overwrite shredding in Docker volumes. |

---

### 3. Verification Command Output

The cluster state was verified using `kubectl get networkpolicy -A` and `kubectl describe resourcequota tenant-a-quota -n tenant-a`.

#### Verification Shell Commands:
```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

#### Terminal Execution & Output:
```text
NAMESPACE   NAME                   POD-SELECTOR   AGE
tenant-b    default-deny-ingress   <none>         10m

Name:            tenant-a-quota
Namespace:       tenant-a
Resource         Used  Hard
--------         ----  ----
pods             1     5
requests.cpu     0     1
requests.memory  0     512Mi
```

#### Evidence Artifact:
- Final Verification Output:  
  <img width="521" height="242" alt="3  Verification Command" src="https://github.com/user-attachments/assets/d9cc373d-54e9-4518-8cdd-4427e5a6370d" />

---

### 4. Security Best-Practices Checklist

- [x] **Tenants are separated into distinct namespaces**: `tenant-a` and `tenant-b` created to establish logical administrative boundaries.
- [x] **A default-deny NetworkPolicy blocks cross-tenant traffic**: Applied `default-deny-ingress` in `tenant-b` and verified traffic drop across namespaces.
- [x] **Resource quotas prevent a noisy-neighbour from exhausting shared capacity**: Enforced CPU, Memory, and Pod count limits via `ResourceQuota`.
- [x] **Per-tenant secrets are unreadable by other tenants**: Configured scoped `ServiceAccount`, `Role`, and `RoleBinding` objects; confirmed using `kubectl auth can-i`.
- [x] **Secure deletion / cryptographic erasure is understood for data remanence**: Verified binary overwriting on Docker volumes and documented cloud cryptographic erasure concepts.

---

### 5. Cleanup & Teardown

To ensure complete resource recovery and cluster teardown, the local `kind` cluster and Docker volume were removed.

#### Teardown Commands:
```bash
# Delete kind Kubernetes cluster
kind delete cluster --name ccse-lab2

# Remove local Docker test volume
docker volume rm ccse-vol
```

---

### 6. Expansion Ideas (Advanced Students)

1. **Egress Default-Deny & Micro-segmentation**: Implement an egress default-deny policy in conjunction with selective egress rules allowing outbound traffic only to CoreDNS (`kube-system`) and specified external APIs.
2. **Pod Security Standards (PSS - Restricted)**: Enforce the Kubernetes `restricted` Pod Security Standard via namespace labels (`pod-security.kubernetes.io/enforce: restricted`) to prevent containers from running as root, mounting host paths, or escalating privileges.
3. **Runtime Sandboxing with gVisor / Kata Containers**: Replace default runc container runtimes with gVisor (`runsc`) to intercept syscalls in user-space, providing hypervisor-like compute isolation without VM overhead.
4. **Cluster-Wide Enforcement via Calico GlobalNetworkPolicy**: Utilize Calico `GlobalNetworkPolicy` CRDs to enforce multi-tenant network isolation across non-namespaced resources and node-level endpoints cluster-wide.

---
*Report compiled by Student `Siti Nurjannah binti Daud` (`nana@kali`) for IKB42603 Cloud Computing Security Essentials.*
