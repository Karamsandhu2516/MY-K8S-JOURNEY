## Day 1: Kubernetes Lab Setup with kubeadm

Before we deploy Pods, expose Services, configure Ingress, manage storage, or troubleshoot production issues, we need a proper Kubernetes environment to learn on.

In Day1 i am going to setup the environment 
By the end of this article, you’ll have:

. Three Ubuntu nodes ready for Kubernetes
. containerd installed and configured
. kubeadm, kubelet, and kubectl installed
. Node-to-node connectivity verified
. Infrastructure ready for cluster initialization

I am going to start with Kubeadm though cloud providers provide managed cluster like EKS, GKS, AKS but understanding of underlaying architechture is must when someone really want to work or pursue career in devops or K8S.

you should know:

How clusters are initialized
How worker nodes join a cluster
How certificates are managed
How control plane components communicate
How upgrades work
How backup and recovery work
How Kubernetes behaves during failures
This is exactly why we’ll use kubeadm.

kubeadm exposes the components and workflows that real administrators interact with every day.  

I am using mutipass for this lab 

Multipass: A lightweight orchestration tool created by Canonical (the makers of Ubuntu) designed to spin up pristine Ubuntu instances instantly using native hypervisors (like HyperKit/Apple Virtualization framework on macOS, Hyper-V on Windows, or KVM on Linux).

## Lab Topology & Node Details
* **Hypervisor:** Multipass (Ubuntu 24.04 LTS)
* **Control Plane:** `k8s-control-plane` | IP: `192.168.252.2` | 2 vCPU | 4 GB RAM
* **Worker Node 1:** `k8s-worker-1` | IP: `192.168.252.3` | 2 vCPU | 2 GB RAM
* **Worker Node 2:** `k8s-worker-2` | IP: `192.168.252.4` | 2 vCPU | 2 GB RAM

## Completed Tasks

### 1. VM Deployment & Hostname Configuration
Created three Ubuntu instances via Multipass and assigned hostnames
- `k8s-control-plane`
- `k8s-worker-1`
- `k8s-worker-2`

### 2. Host Resolution (`/etc/hosts`)
Configured `/etc/hosts` on all nodes to map hostnames to static local IPs[cite: 1]:

### 3. OS Updates & Swap Disabled
- Updated system packages via `apt update && apt upgrade -y`
- Disabled swap memory to ensure predictable container resource allocation

### 4. Kernel Modules & Networking Parameters
Loaded required kernel modules and configured routing parameters for Kubernetes networking


# Note make sure local network allow traffic from Editor or terminal 
### 🚨 Troubleshooting: Multipass `No route to host` on macOS
If you encounter `shell failed: ssh connection failed: 'Failed to connect: No route to host'`, the issue may be related to macOS privacy settings rather than Multipass itself.

**The Fix:**
1. Go to **System Settings** > **Privacy & Security** > **Local Network**.
2. Locate **Visual Studio Code** (or your terminal application like iTerm/Terminal) and ensure it is **toggled ON** to allow finding and communicating with devices on your local network.
3. Restart your Multipass instances if necessary:
   ```bash
   multipass restart --all ```


   
## Lab Task 8: Install Kubernetes Components

The Kubernetes tools are installed across all nodes (`k8s-control-plane`, `k8s-worker-1`, `k8s-worker-2`) using the official Kubernetes package repository:

``` bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.31/deb/](https://pkgs.k8s.io/core:/stable:/v1.31/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

## Lab Task 9: Verify Installation

Check version information for each component across the nodes:
``` bash
kubeadm version
kubectl version --client
kubelet --version 
```

## Lab Task 10 : Verify Node Connectivity

from control-plane to worker-node
```bash
ping -c 3 k8s-worker-1
ping -c 3 k8s-worker-2

From the worker nodes
ping -c 3 k8s-control-plane

Verification: Successful communication (0% packet loss) confirms the network environment is fully prepared for cluster initialization.