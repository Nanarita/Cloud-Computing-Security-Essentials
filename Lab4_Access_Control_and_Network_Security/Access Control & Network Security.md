# IKB42603 Cloud Computing Security Essentials
## Lab 4 Comprehensive Report: Access Control & Network Security (AuthN vs AuthZ, network segmentation and host hardening — Docker & Kubernetes)

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Lab Assignment:** Lab 4 (Weeks 7–8) — Access Control & Network Security  
**Course Learning Outcome:** CLO2 — Construct secure cloud operations that safeguard data integrity  
**Environment User:** `fikri`  

---

## Executive Summary & Learning Outcomes

This step-by-step lab report documents the implementation and evaluation of access control mechanisms and network security principles utilizing Docker and Kubernetes. The lab covers identity verification, role-based permissions, network segmentation, and container hardening techniques.

### Key Objectives & Achievements:
1. **Authentication & Authorization (Session A)**: Distinguished and implemented authentication (verifying identity via HTTP Basic Auth) and authorization (enforcing permissions via Kubernetes RBAC).
2. **Multi-Factor Authentication (MFA)**: Added a second factor of authentication using a Time-based One-Time Password (TOTP) code and validated it to strengthen credential security.
3. **Network Segmentation**: Configured network access control and segmentation (Three-Tier architecture) using Docker networks to ensure services can only reach what they strictly must, enforcing lateral movement constraints.
4. **Firewall Rules**: Applied host-level default-deny firewall policies using `iptables`, mirroring the operational model of cloud security groups.
5. **Container & Host Hardening**: Reduced container attack surfaces by deploying a minimal image as a non-root user, dropping all Linux capabilities, and mounting a read-only root filesystem. Scanned images for vulnerabilities using Trivy.

---

## Table of Contents
1. [Session A (Week 7): Authentication & Authorization](#session-a-week-7-authentication--authorization)
   - [Task 1: Authentication: a Password-Protected Service](#task-1-authentication-a-password-protected-service)
   - [Task 2: Add a Second Factor (MFA / TOTP)](#task-2-add-a-second-factor-mfa--totp)
   - [Task 3: Authorization: RBAC Roles](#task-3-authorization-rbac-roles)
2. [Session B (Week 8): Network Security & Hardening](#session-b-week-8-network-security--hardening)
   - [Task 4: Network Segmentation (Three-Tier)](#task-4-network-segmentation-three-tier)
   - [Task 5: Firewall Rules (Default-Deny)](#task-5-firewall-rules-default-deny)
   - [Task 6: Container / Host Hardening](#task-6-container--host-hardening)
3. [Deliverables & Assessment](#deliverables--assessment)
   - [1. Screenshots Reference Checklist](#1-screenshots-reference-checklist)
   - [2. Short-Answer Questions (Q1 - Q5)](#2-short-answer-questions-q1---q5)
   - [3. Verification Command Output](#3-verification-command-output)
   - [4. Security Best-Practices Checklist](#4-security-best-practices-checklist)
   - [5. Cleanup & Teardown](#5-cleanup--teardown)
   - [6. Expansion Ideas (Advanced Students)](#6-expansion-ideas-advanced-students)

---

## Session A (Week 7): Authentication & Authorization

### Task 1: Authentication: a Password-Protected Service

Authentication proves *who* a user is. In this task, a web service was deployed behind HTTP Basic authentication, ensuring only requests bearing valid credentials (`student` / `P@ssw0rd!`) were granted access.

#### Shell Commands Executed:
```bash
# Create a password file (user: student)
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt

# Serve a page that requires authentication
cat > default.conf <<'EOF'
server { listen 80;
 location / { auth_basic "Restricted";
 auth_basic_user_file /etc/nginx/.htpasswd;
 return 200 'Authenticated OK\n'; } }
EOF

# Run Nginx with auth configuration
docker run --rm -d --name authsvc -p 8080:80 \
 -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
 -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd nginx

# Test access without credentials (Expected: 401 Unauthorized)
curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080

# Test access with credentials (Expected: 200 OK)
curl -s -u student:'P@ssw0rd!' http://localhost:8080
```

#### Terminal Execution & Output:
```text
no-creds: 401
Authenticated OK
```

#### Evidence Artifacts:
- 401 Unauthorized (No Creds): ![Task1-1](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task1.%20%20Authentication_a%20Password-Protected%20Service.PNG)
- 200 OK (Valid Creds): ![Task1-2](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task1.%20Authentication_a%20Password-Protected%20Service.PNG)

---

### Task 2: Add a Second Factor (MFA / TOTP)

Passwords alone are susceptible to theft. This task demonstrated the generation and validation of a time-based one-time password (TOTP), representing the "something you have" factor in MFA.

#### Shell Commands Executed:
```bash
# Create a shared secret (base32) and generate the current 6-digit code
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enrol this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"

# Validate a code the user types (compare to the expected value)
read -p 'Enter the 6-digit code: ' CODE
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```

#### Terminal Execution & Output:
```text
Enrol this secret in an authenticator app: [GENERATED_SECRET]
[GENERATED_CODE]
Enter the 6-digit code: [GENERATED_CODE]
MFA OK
```

> [!TIP]
> MFA combines factors from different classes. It defeats the majority of credential attacks — the cheapest big security win.

#### Evidence Artifact:
- MFA Verification Success: ![Task2](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task2.%20Add%20a%20Second%20Factor.PNG)

---

### Task 3: Authorization: RBAC Roles

While authentication proves identity, authorization decides permissions. A `kind` cluster was established to demonstrate Kubernetes Role-Based Access Control (RBAC), applying the principle of least privilege to a developer service account.

#### Shell Commands Executed:
```bash
kind create cluster --name ccse-lab4
kubectl create namespace app
kubectl create serviceaccount dev -n app

# Developer may only read pods
kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev

SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA       # Expected: yes
kubectl auth can-i create deploy -n app --as=$SA   # Expected: no
kubectl auth can-i delete pods -n app --as=$SA     # Expected: no
```

#### Terminal Execution & Output:
```text
yes
no
no
```

#### Evidence Artifact:
- RBAC Verification (Allowed vs Denied): ![Task3](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task3.%20%20Authorization_RBAC%20Roles.PNG)

---

## Session B (Week 8): Network Security & Hardening

### Task 4: Network Segmentation (Three-Tier)

Network segmentation isolates resources, restricting lateral movement. A simulated three-tier architecture was built where the frontend web tier cannot directly access the database, adding defence in depth.

#### Shell Commands Executed:
```bash
# Create two segmented networks
docker network create frontend-net
docker network create backend-net

# DB only on backend-net; app on both; web only on frontend-net
docker run -d --name db --network backend-net redis:alpine
docker run -d --name app --network backend-net nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx

# web -> db should FAIL (not on the same network)
docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'

# app -> db should WORK (shared backend-net)
docker exec app sh -c 'apt-get update -qq && apt-get install -y -qq netcat-openbsd && nc -z -w3 db 6379 && echo REACHABLE'
```

#### Terminal Execution & Output:
```text
BLOCKED
REACHABLE
```

> [!NOTE]
> The database is unreachable from the internet-facing tier. An attacker who compromises the web tier still cannot talk directly to the data — segmentation contains lateral movement.

#### Evidence Artifacts:
- Segmentation Block & Allow:
  - ![Task4-1](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task4.%20%20Network%20Segmentation%20(Three-Tier).PNG)
  - ![Task4-2](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task4.%20Network%20Segmentation%20(Three-Tier).PNG)

---

### Task 5: Firewall Rules (Default-Deny)

A default-deny firewall policy mirrors cloud security group models by ensuring nothing is allowed unless explicitly permitted (least privilege for the network).

#### Shell Commands Executed:
```bash
# Inside a throwaway container with iptables, model default-deny + allow 443
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
 apk add -q iptables; \
 iptables -P INPUT DROP; \
 iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
 iptables -A INPUT -i lo -j ACCEPT; \
 iptables -L INPUT -n'
```

#### Evidence Artifact:
- Default-Deny Iptables Ruleset: ![Task5](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task5.%20Firewall%20Rules%20(Default-Deny).PNG)

---

### Task 6: Container / Host Hardening

To drastically reduce the attack surface, a container was deployed using non-root privileges, a read-only filesystem, dropped Linux capabilities, and no-new-privileges flags. The base image was then scanned for vulnerabilities.

#### Shell Commands Executed:
```bash
# A hardened run of a service
docker run -d --name hardened \
 --user 1000:1000 \
 --read-only \
 --cap-drop=ALL \
 --security-opt no-new-privileges \
 --tmpfs /tmp \
 nginxinc/nginx-unprivileged

# Verify the hardening settings were actually applied
docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'

# Scan an image for known vulnerabilities
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

#### Terminal Execution & Output:
```text
User=1000:1000 ReadOnly=true
[Trivy Vulnerability Scan Output...]
```

#### Evidence Artifacts:
- Container Inspect & Trivy Scans:
  - ![Task6-1](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%20Container_Host%20Hardening.PNG)
  - ![Task6-2](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%202Container_Host%20Hardening.PNG)
  - ![Task6-3](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%203Container_Host%20Hardening.PNG)
  - ![Task6-4](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%204Container_Host%20Hardening.PNG)
  - ![Task6-5](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%205Container_Host%20Hardening.PNG)

---

## Deliverables & Assessment

### 1. Screenshots Reference Checklist

| Task | Description | Evidence File Path |
|---|---|---|
| **Task 1** | 401 and 200 status results | [`Lab4-Evidence/Task1.  Authentication_a Password-Protected Service.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task1.%20%20Authentication_a%20Password-Protected%20Service.PNG) |
| **Task 2** | MFA OK output for valid TOTP | [`Lab4-Evidence/Task2. Add a Second Factor.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task2.%20Add%20a%20Second%20Factor.PNG) |
| **Task 3** | Three `auth can-i` RBAC results | [`Lab4-Evidence/Task3.  Authorization_RBAC Roles.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task3.%20%20Authorization_RBAC%20Roles.PNG) |
| **Task 4** | web→db BLOCKED / app→db REACHABLE | [`Lab4-Evidence/Task4.  Network Segmentation (Three-Tier).PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task4.%20%20Network%20Segmentation%20(Three-Tier).PNG) |
| **Task 5** | iptables default-deny ruleset | [`Lab4-Evidence/Task5. Firewall Rules (Default-Deny).PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task5.%20Firewall%20Rules%20(Default-Deny).PNG) |
| **Task 6** | Hardened container inspect & Trivy scan | [`Lab4-Evidence/Task6. Container_Host Hardening.PNG`](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/Task6.%20Container_Host%20Hardening.PNG) |

---

### 2. Short-Answer Questions (Q1 - Q5)

#### **Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.**
- **Authentication (Task 1)** focuses on verifying identity—proving *who* you are. We established this by enforcing HTTP Basic Auth, where a valid username (`student`) and password combination successfully verified our identity.
- **Authorization (Task 3)** determines *permissions*—what you are allowed to do after authentication. This was shown using Kubernetes RBAC. The `dev` service account was authenticated, but its authorization role only explicitly permitted reading pods, effectively denying deployment creation or pod deletion requests.

#### **Q2. Why is MFA so effective, and which attacks does it defeat?**
Multi-Factor Authentication (MFA) requires users to provide two or more distinct types of evidence to log in (e.g., something you know like a password, and something you have like a time-based TOTP code). It is highly effective because it neutralizes the vast majority of credential-based attacks—such as phishing, password spraying, and brute force—since an attacker cannot gain access with stolen credentials alone without possessing the physical secondary token device.

#### **Q3. How does network segmentation limit the damage of a compromised web server?**
Network segmentation divides infrastructure into isolated zones (such as `frontend-net` and `backend-net`). If a public-facing web server in the frontend network is breached, the segmentation acts as a containment boundary. The attacker's lateral movement is halted because there is no direct network routing path or shared Layer 2 connection that allows the compromised server to talk directly to the sensitive backend database.

#### **Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?**
A default-deny policy (like `iptables -P INPUT DROP`) ensures that all network traffic is blocked by default, and only explicitly permitted ports (like port 443) are allowed through. This applies the principle of least privilege to networking. Cloud Security Groups (like those in AWS) operate on this exact model: they inherently block all inbound traffic unless an administrator creates an explicit allow rule.

#### **Q5. List the hardening measures you applied and the attack surface each one removes.**
1. **`--user 1000:1000` (Non-root user):** Mitigates privilege escalation; a compromised container process runs as an unprivileged user, protecting the underlying host node.
2. **`--read-only` (Read-only rootfs):** Prevents malware persistence and code tampering by making the container filesystem immutable. Attackers cannot install packages or modify system binaries.
3. **`--cap-drop=ALL` (Drop Linux capabilities):** Massively shrinks the kernel attack surface by revoking powerful but unnecessary OS privileges (like raw packet sniffing, system clock manipulation, or kernel module loading).
4. **`--security-opt no-new-privileges`:** Prevents existing processes from gaining new privileges (e.g., stopping an attacker from executing a `setuid` binary to escalate privileges).
5. **`--tmpfs /tmp`:** Maps an ephemeral, RAM-backed volume strictly for temporary files, preventing an attacker from writing executable malware payloads to persistent disk storage.

---

### 3. Verification Command Output

The environment's RoleBindings and container dropped capabilities were verified.

#### Shell Commands Executed:
```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

#### Evidence Artifact:
- Verification Execution: ![3. Verification Command.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/3.%20Verification%20Command.PNG)

---

### 4. Security Best-Practices Checklist

- [x] Service requires authentication (unauthenticated requests rejected).
- [x] MFA / second factor implemented and validated.
- [x] Authorization enforced by RBAC (least privilege; unauthorised actions denied).
- [x] Network segmented so the data tier is unreachable from the front tier.
- [x] Default-deny firewall with explicit allow rules.
- [x] Container hardened: non-root, minimal, capabilities dropped, read-only; image scanned.

---

### 5. Cleanup & Teardown

To ensure complete resource recovery, containers, networks, and the cluster were removed.

#### Teardown Commands:
```bash
docker rm -f authsvc db app web hardened 2>/dev/null
docker network rm frontend-net backend-net 2>/dev/null
kind delete cluster --name ccse-lab4
```

#### Evidence Artifact:
- Cleanup Execution: ![CleanUp and TearDown.PNG](file:///c:/Users/fikri/Documents/Sem%206%20Short%20Sem/Cloud%20Computing/Lab4-Evidence/CleanUp%20and%20TearDown.PNG)

---

### 6. Expansion Ideas (Advanced Students)

- Add a **Web Application Firewall (ModSecurity)** in front of the service and block a test SQL-injection string.
- Deploy **fail2ban** to auto-block IPs after repeated failed logins.
- Introduce a service mesh (**Istio**) and enforce **mTLS** between services (zero-trust network).
- Turn the hardening steps into a **Dockerfile** with a distroless base image and rebuild.

---
*Report compiled by Student `fikri` for IKB42603 Cloud Computing Security Essentials.*
