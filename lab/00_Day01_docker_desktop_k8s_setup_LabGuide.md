# Lab Guide: Docker Desktop Kubernetes Setup

**Target Audience:** Trainees with Windows, Mac, or Linux machines  
**Duration:** 45–60 minutes  
**Goal:** Setup a local Kubernetes cluster using Docker Desktop for hands-on practice

---

## Overview

This lab guide walks you through setting up a local Kubernetes cluster using **Docker Desktop**, which includes Kubernetes as an built-in feature. You'll verify the setup and run your first pod.

### Why Docker Desktop Kubernetes?
- Free, easy to install
- Single-node cluster (perfect for learning)
- Works on Windows, Mac, and Linux
- Includes container registry, Docker CLI, and Kubernetes

### What You'll Build
- Docker Desktop installed with Kubernetes enabled
- `kubectl` configured and connected
- First pod deployed and verified

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Windows 10 64-bit, macOS 10.15+, Ubuntu 22.04+ | Windows 11, macOS 12+, Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 25 GB free | 50 GB free |
| Docker Version | 20.10+ | 24.0+ |
| Virtualization | Enabled (VT-x/AMD-V) | Enabled (VT-x/AMD-V) |

### Before You Start
- Administrative access on your machine
- Internet connection (for downloads)
- No conflicting Docker installations (check if Docker is already installed)

---

## Step 1: Install Docker Desktop

### Windows

1. **Download Docker Desktop**
   - Go to: https://www.docker.com/products/docker-desktop/
   - Click **Download for Windows**
   - Save `Docker-Desktop-Win-x86_x64.exe`

2. **Run Installer**
   ```bash
   # Double-click Docker-Desktop-Win-x86_x64.exe
   # Accept UAC prompt if shown
   ```

3. **Configure WSL 2 Backend**
   - Docker will prompt to use WSL 2 backend
   - Click **Use WSL 2 instead of Hyper-V**
   - If WSL 2 not installed:
     ```powershell
     # Open PowerShell as Administrator
     wsl --install
     # Restart machine, then run:
     wsl --set-version Ubuntu 2
     ```

4. **Complete Installation**
   - Click **OK** to finish
   - Docker Desktop starts automatically

5. **Accept Docker Subscription Agreement**
   - On first launch, accept the terms

### macOS

1. **Download Docker Desktop**
   - Go to: https://www.docker.com/products/docker-desktop/
   - Click **Download for Mac**
   - Save `Docker.dmg`

2. **Install Docker**
   ```bash
   # Double-click Docker.dmg
   # Drag Docker to Applications folder
   ```

3. **Start Docker Desktop**
   - Open **Applications → Docker**
   - Click **Open** if security prompt appears
   - Docker starts in background

4. **Grant Permissions (if needed)**
   - Go to **System Preferences → Security & Privacy**
   - Allow Docker if blocked

5. **Accept Docker Subscription Agreement**
   - On first launch, accept the terms

### Linux (Ubuntu)

1. **Confirm KVM Availability**
   ```bash
   # Check if /dev/kvm exists
   ls -l /dev/kvm

   # Check if kvm module is loaded
   lsmod | grep kvm

   # Check if user is in kvm group
   groups | grep kvm

   # If any check fails:
   # Missing /dev/kvm: Enable virtualization in BIOS (Intel VT-x or AMD-V)
   # Module not loaded: sudo modprobe kvm_intel (or kvm_amd)
   # Not in group: sudo usermod -aG kvm $USER, then log out and back in
   ```

2. **Set up Docker Repository**
   ```bash
   # Update package index
   sudo apt-get update

   # Install prerequisites
   sudo apt-get install -y ca-certificates curl gnupg

   # Create Docker directory
   sudo mkdir -p /etc/apt/keyrings

   # Download Docker GPG key
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg |      sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

   # Add Docker repository
   echo "deb [arch=$(dpkg --print-architecture)      signed-by=/etc/apt/keyrings/docker.gpg]      https://download.docker.com/linux/ubuntu      $(lsb_release -cs) stable" |      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. **Install Docker Desktop**
   ```bash
   # Update package index
   sudo apt-get update

   # Download and install DEB package
   sudo apt-get install -y ./docker-desktop-amd64.deb
   ```

4. **Launch Docker Desktop**
   ```bash
   # Start from application launcher:
   # Navigate to Gnome/KDE Desktop → Applications → Docker Desktop

   # Or start from terminal:
   systemctl --user start docker-desktop
   systemctl --user enable docker-desktop
   ```

5. **Accept Docker Subscription Agreement**
   - On first launch, accept the terms

6. **Switch to Docker Desktop Context (if Docker Engine installed)**
   ```bash
   # List contexts
   docker context ls

   # Switch to Docker Desktop context
   docker context use desktop-linux
   ```

---

## Step 2: Enable Kubernetes in Docker Desktop

### Windows & macOS & Linux

1. **Open Docker Settings**
   - Click **Docker icon** in taskbar/menu bar
   - Go to **Settings → Kubernetes** (or **Preferences → Kubernetes** on macOS)

2. **Enable Kubernetes**
   ```
   ☑ Enable Kubernetes
   ```

3. **Apply & Restart**
   - Click **Apply & Restart**
   - Click **Install** to confirm
   - Docker restarts (takes 2–5 minutes)

4. **Wait for Kubernetes to Start**
   - Docker icon shows **Kubernetes running**
   - Status: `Kubernetes is running`

> **Note:** Docker Desktop does not upgrade your Kubernetes cluster automatically after a new update. To upgrade, select **Reset Kubernetes Cluster**.

---

## Step 3: Install & Configure kubectl

### Windows

```bash
# Option 1: Direct download via curl
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
Move-Item "kubectl.exe" "$env:USERPROFILE\.bin\kubectl.exe"

# Add to PATH (if not already)
$env:PATH += ";$env:USERPROFILE\.bin"
[Environment]::SetEnvironmentVariable("PATH", $env:PATH, "User")

# Option 2: Install via winget
winget install -e --id Kubernetes.kubectl

# Option 3: Install via Chocolatey
choco install kubernetes-cli

# Verify installation
kubectl version --client
```

> **Note:** Docker Desktop for Windows adds its own `kubectl` to PATH. If you installed Docker Desktop before, you may need to place your PATH entry before the one added by Docker Desktop or remove Docker Desktop's kubectl.

### macOS

```bash
# Option 1: Install via Homebrew
brew install kubectl

# Option 2: Download manually
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/mac/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

> **Note:** If you installed kubectl using Homebrew and experience conflicts, remove `/usr/local/bin/kubectl` and use Docker Desktop's version.

### Linux

```bash
# Download and install kubectl
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

> **Note:** The kubectl binary is not automatically packaged with Docker Desktop for Linux. You must install it separately.

---

### Configure kubectl to Use Docker Desktop Cluster

```bash
# Check available contexts
kubectl config get-contexts

# Expected output:
# NAME             CLUSTER          AUTHINFO         NAMESPACE
# docker-desktop   docker-desktop   docker-desktop   default

# Switch to docker-desktop context (if needed)
kubectl config use-context docker-desktop

# Verify cluster connection
kubectl cluster-info

# Expected: URL response showing cluster info
```

> **Note:** Run `kubectl` in CMD or PowerShell on Windows (not WSL), otherwise `kubectl config get-contexts` may return empty results.

---

## Step 4: Verify Setup

### 1. Check Nodes

```bash
kubectl get nodes
```

**Expected:**
```
NAME           STATUS   ROLES    AGE   VERSION
docker-desktop Ready    master   5m    v1.28.0
```

### 2. Check System Pods

```bash
kubectl get pods -n kube-system
```

**Expected:**
```
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxx                       1/1     Running   0          5m
kube-apiserver-docker-desktop          1/1     Running   0          5m
kube-controller-manager-docker-desktop 1/1     Running   0          5m
kube-proxy-xxxxxxxx                    1/1     Running   0          5m
kube-scheduler-docker-desktop          1/1     Running   0          5m
```

### 3. Test with Sample Pod

```bash
# Deploy nginx
kubectl run nginx --image=nginx:1.24 --port=80

# Check pod status
kubectl get pods

# Expected:
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          30s
```

### 4. Expose as Service

```bash
# Create service
kubectl expose nginx --port=80 --type=NodePort

# Get service URL
kubectl get service nginx

# Expected:
# NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
# nginx   NodePort   10.103.25.1   <none>        80/TCP    1m
```

### 5. Access Pod

```bash
# Get pod IP
kubectl get pod nginx -o wide

# Curl the pod
kubectl exec nginx -- curl -s http://localhost:80

# Expected: HTML output from nginx
```

---

## Troubleshooting

### Kubernetes Won't Start

```bash
# Check Docker status
docker ps

# Restart Docker Desktop
# From Docker menu: Restart

# Force enable Kubernetes
# Settings → Kubernetes → Enable → Apply & Restart
```

### kubectl Not Found

```bash
# Windows: Add to PATH
$env:PATH += ";$env:USERPROFILE\.bin"

# macOS/Linux: Check installation
ls -l /usr/local/bin/kubectl

# If not found, reinstall kubectl
```

### Pod Stuck in Pending

```bash
# Check events
kubectl describe pod nginx

# Check node resources
kubectl top nodes

# Common causes:
# - Insufficient CPU/memory
# - No available nodes
# - Image pull error
```

### Image Pull Error

```bash
# Check image in Docker
docker pull nginx:1.24

# Try with explicit registry
kubectl run test --image=docker.io/nginx:1.24
```

### Connection Refused

```bash
# Check cluster info
kubectl cluster-info

# If error shown:
# kubectl is not configured correctly or cannot connect to cluster
# Check if Docker Desktop is running
# Check if context is set to docker-desktop
kubectl config use-context docker-desktop
```

---

## Cleanup

```bash
# Delete pod
kubectl delete pod nginx

# Delete service
kubectl delete service nginx

# Reset cluster (optional)
# Docker Desktop → Settings → Kubernetes → Disable → Apply & Restart
```

---

## Alternative: Pre-Provisioned Cluster

If you cannot install Docker Desktop locally (e.g., restricted environment, old hardware), an alternative is to use a **pre-provisioned cluster** provided by the trainer:

- Trainer will share cluster access credentials
- You'll connect via `kubectl` using provided config
- No local installation required
- Same hands-on exercises apply

Ask your trainer for access details if needed.

---

## Next Steps

After setup, you're ready for:
1. **Session 1 Hands-On:** Cluster navigation
2. **Practice:** Deploy your first application
3. **Explore:** `kubectl get all -A`

---

## Trainer Notes

### Pre-Lab Checklist
- [ ] Verify Docker Desktop version ≥ 24.0
- [ ] Ensure Kubernetes is enabled in Docker settings
- [ ] Test `kubectl cluster-info` on trainer machine
- [ ] Prepare backup SSH access for remote troubleshooting
- [ ] Have pre-provisioned cluster ready as alternative

### Common Issues by Platform

| Platform | Issue | Fix |
|---|---|---|
| Windows | WSL 2 not installed | `wsl --install` |
| macOS | Docker blocked by security | System Preferences → Security & Privacy |
| Linux | Permission denied | `sudo usermod -aG docker $USER` |
| Linux | KVM not available | Enable virtualization in BIOS |
| All | Image pull failed | `docker pull` first |

### Time Estimate

| Step | Duration |
|---|---|
| Install Docker | 15 min |
| Enable Kubernetes | 5 min |
| Install kubectl | 5 min |
| Verify Setup | 10 min |
| Troubleshooting | 10–15 min |
| **Total** | **45–60 min** |

### Notes from Official Documentation

- Docker Desktop includes standalone Kubernetes server (single-node, for local testing only) [web:19]
- Kubernetes is not configurable and runs within a Docker container [web:19]
- kubectl is installed at `/usr/local/bin/kubectl` on Mac, Docker adds it to PATH on Windows [web:19]
- Docker Desktop does not upgrade Kubernetes automatically—use **Reset Kubernetes Cluster** [web:19]
- kubectl version must be within one minor version of cluster (e.g., v1.28 client with v1.27–v1.29 cluster) [web:20]

---

**End of Lab Guide**
