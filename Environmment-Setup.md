# Environment Setup — IKB42603 Lab0

**Name:** Siti Nurjannah binti Daud
**Student ID:** 52215124446
**Date:** 29 July 2026

## 1) Install Docker
- Download & install Docker Desktop for Windows and enable WSL2 or Hyper-V as prompted.
- Start Docker Desktop and ensure it is running.
  ```powershell
  docker --version
  ```

 Evidence:
 
 <img width="351" height="68" alt="1  Docker" src="https://github.com/user-attachments/assets/f8cfa18e-b498-4260-80e9-d9ac5c9fc8eb" />


## 2) Install AWS CLI v2
- Download the AWS CLI v2 MSI for Windows and run the installer.
- Verify:
  ```powershell
  aws --version
  ```
 Evidence:
 
 <img width="624" height="67" alt="2  AWS CLI v2" src="https://github.com/user-attachments/assets/884ae615-60e5-4535-bf21-33d0a6a7e6d9" />

## 3) Install kind and kubectl
- Download the kind Windows binary or use Go if available. Example with curl (PowerShell):
  ```powershell
  curl -Lo kind.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
  chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
  kind --version
  ```
  
 Evidence:
 
 <img width="199" height="63" alt="3  Kind" src="https://github.com/user-attachments/assets/1e2a20dc-661d-4cf8-91de-bdacf2cba18c" />

- Download the kubectl binary for Windows and place it in a directory on your PATH.
- Verify kubectl:
  ```powershell
  kubectl version --client
  ```
  
 Evidence:
 
 <img width="266" height="73" alt="3  Kubectl" src="https://github.com/user-attachments/assets/cb20e6e7-e3c3-449b-9e98-2e365dbbec1d" />

## 4) Install Helper Tools
- OpenSSL (for certs): install it from the official Windows build or Git for Windows bundle, then verify:
  ```powershell
  openssl version
  ```
  
 Evidence:
 
 <img width="518" height="64" alt="4  OpenSSL" src="https://github.com/user-attachments/assets/cee6d5e8-f8ab-44d2-b053-6f4565f995ef" />

  ```powershell
  oathtool --version
  ```

 Evidence:
 
 <img width="657" height="157" alt="4  Oathtool" src="https://github.com/user-attachments/assets/3edf520a-c806-40a4-8954-7afb1ece490e" />

5) Starts Local Stack
- Install LocalStack (for AWS service emulation) using pip or Docker:
  ```powershell
  sudo docker run --rm -p 4566:4566 localstack/localstack:3.0
  localstack --version
  ```
- Or run via Docker Compose as shown in the guide.
  
 Evidence:

 <img width="1137" height="692" alt="5  LocalStack" src="https://github.com/user-attachments/assets/351abcae-7dc0-4298-aec4-3e0b772d7e13" />

## 6) Create the Kubernetes cluster with kind
- Create the cluster:
  ```powershell
  kind create cluster --name ccse
  kubectl cluster-info --context kind-ccse
  kubectl get nodes
  ```
  
 Evidence:
 
 <img width="519" height="81" alt="5  Kubernetes cluster KIND" src="https://github.com/user-attachments/assets/5c67ef58-0098-4e58-915a-2e7a62ed775a" />

## 7) Final checks
- The cheatsheet requires dummy credentials because LocalStack accepts arbitrary values. Set dummy values once so the CLI stops asking:
  ```powershell
  aws configure set aws_access_key_id test
  aws configure set aws_secret_access_key test
  aws configure set region us-east-1
  ```
  ```powershell
  EP='--endpoint-url=http://localhost:4566'
  aws $EP sts get-caller-identity
  ```
  
 Evidence:
 
 <img width="384" height="126" alt="6  One time" src="https://github.com/user-attachments/assets/5edda029-ccc6-4ca3-93ab-e4d88b769aa4" />

