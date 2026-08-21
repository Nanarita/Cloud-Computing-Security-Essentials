# IKB42603 Cloud Computing Security Essentials

### Student Information
* **Name:** Siti Nurjannah binti Daud
* **Student ID:** 52215124446
* **Course:** IKB42603 Cloud Computing Security Essentials
* **Lecturer:** Adani binti Kamal

---

### Course Summary
This repository contains my lab reports, shell executions, and coursework for the Cloud Computing Security Essentials course[cite: 1]. The primary learning outcome of this course is to construct secure cloud operations that safeguard data integrity[cite: 3]. 

Throughout the labs, we explore fundamental cloud security architecture by simulating cloud environments locally. Key topics covered include:
* **Identity & Access Management (IAM):** Applying the principle of least privilege, managing identity governance, and implementing Role-Based Access Control (RBAC)[cite: 3].
* **Secure Isolation & Multi-Tenancy:** Enforcing compute, network, and storage isolation to prevent cross-tenant access using namespaces, resource quotas, and default-deny network policies[cite: 4].
* **Data Protection & Cryptography:** Managing data at rest and in transit using symmetric/asymmetric encryption, implementing envelope encryption with Key Management Services (KMS), and ensuring provable deletion through cryptographic erasure[cite: 5].

---

### Lab Environment & Tooling
All lab simulations and tasks in this repository were executed locally inside a **Kali Linux** virtual machine running on **VMware**. To ensure a secure, self-contained testing environment, the labs run completely offline without connecting to a real commercial cloud provider[cite: 3].

Below is the technology stack utilized throughout the labs:

| Tool | Purpose / Function |
| :--- | :--- |
| **Docker** | Runs containers and hosts the LocalStack cloud simulator[cite: 1]. |
| **LocalStack** | An AWS-compatible cloud simulator used to emulate AWS APIs (like IAM, KMS, S3, and DynamoDB) locally[cite: 2, 3]. |
| **AWS CLI v2** | Command-line interface used to send AWS commands directly to the LocalStack endpoints[cite: 1]. |
| **kind** | Short for "Kubernetes-in-Docker," used to run a local Kubernetes cluster for container orchestration[cite: 1]. |
| **kubectl** | The primary command-line tool used to control and interact with the Kubernetes cluster[cite: 1]. |
| **Calico** | A Container Network Interface (CNI) installed on the cluster to act as a firewall and enforce network security policies[cite: 4]. |
| **OpenSSL** | Used for generating keys, creating digital certificates, and performing symmetric/asymmetric encryption[cite: 1, 5]. |
| **oathtool** | Command-line utility used to generate MFA / TOTP authentication codes[cite: 1]. |
| **Trivy** | A security scanner used to check container images for vulnerabilities[cite: 1]. |
