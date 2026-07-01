# GitOps Setup with KinD and ArgoCD

This repository provides a step-by-step guide to setting up a local Kubernetes cluster using **KinD (Kubernetes in Docker)** on an AWS EC2 instance and deploying **ArgoCD** for GitOps continuous delivery.

---

## Infrastructure Prerequisites

Before running the installation scripts, provision an AWS EC2 instance with the following minimum specifications to prevent Kubernetes or Docker from crashing:

* **AMI:** Ubuntu Server 24.04 LTS (64-bit)
* **Instance Type:** `t3.medium` (Minimum 2 vCPUs and 4 GiB RAM required)
* **Storage:** 30 GB gp3 SSD
* **Security Group Rules:**
  * **SSH (Port 22):** Restrict to your public IP.
  * **Custom TCP (Port 8080):** Open to `0.0.0.0/0` (Anywhere) for accessing the ArgoCD UI dashboard.

---

## Installation & Setup Steps

SSH into your EC2 instance and execute the following phases sequentially.

### Phase 1: Update and Prepare System Dependencies
Update your local package indexes and install fundamental networking/certificate utilities.

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl apt-transport-https ca-certificates gnupg
```
---

### Phase 2: Install and Configure Docker
KinD requires Docker to spin up Kubernetes nodes as containers.

```bash
# 1. Install Docker components
sudo apt-get update
sudo apt-get install -y docker.io

# 2. Start and enable the Docker daemon
sudo systemctl start docker
sudo systemctl enable docker

# 3. Grant your current 'ubuntu' user permission to use Docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```
### Phase 3: Install KinD & Kubectl
Download and install the Kubernetes command-line tool (kubectl) and the KinD binary, then provision the cluster.

```bash
# 1. Download and install Kubectl
curl -LO "[https://dl.k8s.io/release/$](https://dl.k8s.io/release/$)(curl -L -s [https://dl.k8s.io/release/stable.txt](https://dl.k8s.io/release/stable.txt))/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 2. Download and install KinD (Stable AMD64 binary for Linux)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 3. Create your KinD Cluster
kind create cluster --name demo-argocd
```
Verify Cluster Status
Ensure your cluster is fully operational by running:

```bash
kubectl get nodes
```
Note: You should see a node named demo-argocd-control-plane in a Ready status.

### Phase 4: Deploy ArgoCD
Create a dedicated namespace and apply the official stable ArgoCD manifests.

```bash
# 1. Create the namespace
kubectl create namespace argocd

# 2. Apply manifests
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# 3. Monitor the deployment progress until all pods are 'Running'
kubectl get pods -n argocd --watch
```
Accessing the ArgoCD Dashboard
1. Port-Forward to Public Network
Expose the internal argocd-server service to your public EC2 interface via port 8080.

```bash
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:443 > /dev/null 2>&1 &
```
(Note: An & has been added to the end of this command to run it in the background of your terminal session).

2. Retrieve the Initial Admin Password
ArgoCD generates a secure default password for the admin user during installation. Fetch and decode it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
3. Log In
  
. Open your web browser and navigate to: http://<YOUR_EC2_PUBLIC_IP>:8080
   Username: admin
   Password: Use the string output retrieved from the command above.
