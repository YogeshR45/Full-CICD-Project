# Full CI/CD Pipeline using Jenkins, Docker, GitHub Webhooks, and K3s

## Overview

This project demonstrates a **complete end-to-end CI/CD pipeline** using industry-standard DevOps tooling. The pipeline automatically builds, pushes, and deploys a containerized web application to a **K3s Kubernetes cluster** whenever code is pushed to GitHub.

The setup follows a **decoupled architecture**:

* One EC2 instance dedicated to **CI/CD (Jenkins worker plane)**
* One EC2 instance dedicated to **Kubernetes (K3s control plane)**

This mirrors real-world production-grade CI/CD and Kubernetes separation.

---

## Architecture

📸 **Architecture Diagram Placeholder**
Place the architecture diagram image here:

```
snaps/architecture.png
```

---

## Infrastructure Setup

### EC2 Instance 1 – Jenkins / CI Worker Plane

This instance is provisioned using EC2 **user data** and acts as the CI/CD execution environment.

Installed components:

* Jenkins
* Docker
* kubectl
* Java 17 (Jenkins-supported LTS)
* 4 GB swap (to prevent memory pressure)

#### User Data – Jenkins / Worker Plane

```bash
#!/bin/bash
set -e

# -----------------------------
# 1. System update
# -----------------------------
apt-get update -y
apt-get upgrade -y

# -----------------------------
# 2. Create 4GB Swap
# -----------------------------
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# -----------------------------
# 3. Install Docker
# -----------------------------
apt-get install -y docker.io
systemctl enable docker
systemctl start docker

# -----------------------------
# 4. Install Java 17 (Jenkins-supported LTS)
# -----------------------------
apt-get install -y openjdk-17-jdk

# -----------------------------
# 5. Install Jenkins
# -----------------------------
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

apt-get update -y
apt-get install -y jenkins

systemctl enable jenkins
systemctl start jenkins

# -----------------------------
# 6. Add Jenkins to Docker group
# -----------------------------
usermod -aG docker jenkins
systemctl restart jenkins

# -----------------------------
# 7. Install kubectl (latest stable)
# -----------------------------
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl

# -----------------------------
# 8. Final verification log
# -----------------------------
java -version
docker --version
kubectl version --client
free -h
```

Purpose:

* Clone GitHub repository
* Build Docker images
* Push images to DockerHub
* Deploy workloads to K3s

---

### EC2 Instance 2 – K3s Control Plane

This instance hosts the Kubernetes control plane using **K3s**.

Installed components:

* K3s (single-node cluster)
* 4 GB swap

K3s API Server:

```
https://172.31.79.253:6443
```

#### User Data – K3s Control Plane

```bash
#!/bin/bash
set -e

# -----------------------------
# 1. System update
# -----------------------------
apt-get update -y
apt-get upgrade -y

# -----------------------------
# 2. Create 4GB Swap
# -----------------------------
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# -----------------------------
# 3. Install k3s (single-node)
# -----------------------------
curl -sfL https://get.k3s.io | sh -

# -----------------------------
# 4. Verification (logs only)
# -----------------------------
kubectl get nodes
free -h
```

Purpose:

* Run Kubernetes workloads
* Expose application via NodePort

---

## CI/CD Pipeline Flow (Chronological)

### 1. GitHub Webhook Trigger

* Any push to the `main` branch triggers Jenkins automatically
* Event-driven (no polling)

📸 **Screenshot Placeholder**

```
snaps/github-webhook.png
```

---

### 2. Jenkins Pipeline Checkout

Jenkins clones the repository using a **GitHub Personal Access Token (PAT)**.

📸 **Screenshot Placeholder**

```
snaps/jenkins-login.png
snaps/jenkins-checkout.png
```

---

### 3. Docker Image Build

Dockerfile used:

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Image tag strategy:

```
yogeshdock45/drepo:<BUILD_NUMBER>
```

---

### 4. DockerHub Authentication

Jenkins authenticates securely using stored credentials.

Credential:

* `dockerhub-creds` (username + PAT)

---

### 5. Docker Image Push

The built image is pushed to DockerHub.

---

### 6. Kubernetes Manifest Update

The deployment manifest is dynamically updated to reference the new image tag, ensuring immutable deployments and traceability.

---

### 7. Deployment to K3s Cluster

Jenkins authenticates to the K3s cluster using a kubeconfig file.

Credential:

* `k3s-config`

The kubeconfig points to:

```
server: https://172.31.79.253:6443
```

---

## Jenkins Credentials Configuration

Configured credentials:

| Credential ID     | Type           | Purpose            |
| ----------------- | -------------- | ------------------ |
| `github-creds`    | PAT            | GitHub access      |
| `dockerhub-creds` | Username + PAT | DockerHub push     |
| `k3s-config`      | File           | K3s cluster access |

📸 **Screenshot Placeholder**

```
snaps/jenkins-credentials.png
```

---

## Jenkins Plugins

Required plugins (example set):

* Git
* GitHub Integration
* Pipeline
* Credentials Binding
* Docker Pipeline

📸 **Screenshot Placeholder**

```
snaps/jenkins-plugins.png
```

---

## Application Access (Final Output)

The application is exposed via **NodePort** from the K3s cluster.

Access pattern:

```
http://<K3S_NODE_PUBLIC_IP>:<NODEPORT>
```

📸 **Screenshot Placeholders**

```
snaps/k3s-pods.png
snaps/browser-nodeport.png
```

---

## End-to-End Outcome

✔ Code push to GitHub
✔ Jenkins pipeline auto-triggered via webhook
✔ Docker image built and pushed
✔ Kubernetes deployment updated
✔ Application accessible from browser

This project demonstrates:

* Fully automated CI/CD
* Secure credential handling
* Kubernetes deployment lifecycle
* Production-aligned DevOps architecture

---

## Notes

* All provisioning is handled via cloud-init
* No manual steps after initial setup
* Reproducible and interview-ready
* Clean separation of CI and cluster responsibilities
