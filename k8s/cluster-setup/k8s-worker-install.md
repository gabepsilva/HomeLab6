# Kubernetes Worker Node Installation Guide (v1.34.1)



### 1. Update the System

```bash
ssh ubuntu@k8sm1
sudo apt-get update
sudo apt-get upgrade -y
```

### 2. Install Container Runtime (containerd)

Kubernetes needs a container runtime to run containers. We'll install containerd.

```bash
# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg nfs-common

# Install containerd
sudo apt-get install -y containerd

# Create the containerd configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration and save to config file
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Update the configuration to use systemd cgroup driver
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3. Configure System Settings for Kubernetes

```bash
# Disable swap (Kubernetes requirement)
sudo swapoff -a
# Ensure swap remains off after reboot
sudo sed -i '/swap/d' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters without reboot
sudo sysctl --system
```

### 3.1 Add ZFS pv folder
```bash

sudo mkdir /pv
echo '' >> /etc/fstab
echo 'k8s-pv /pv virtiofs defaults,nofail 0 0' >> /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown ubuntu:ubuntu /pv

```



### 4. Install Kubernetes Components

```bash
# Add Kubernetes apt repository
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Download the public signing key for Kubernetes
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index with the new repository
sudo apt-get update

# Install specific Kubernetes version (1.32.3)
sudo apt-get install -y kubelet=1.34.1-1.1 kubeadm=1.34.1-1.1 kubectl=1.34.1-1.1

# Prevent automatic updates for these packages
sudo apt-mark hold kubelet kubeadm kubectl
```

### 5. Join the Worker Node to the Master

On the master node (k8sm1), generate a join command:

```bash
sudo kubeadm token create --print-join-command
```

This will output a command similar to:

```
kubeadm join 10.10.0.148:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

Run the generated command on the worker node with sudo:

```bash
sudo kubeadm join 10.10.0.148:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Verification

After joining the worker node to the cluster, verify from the master node:

```bash
kubectl get nodes
```

