# Kubernetes 1.35 Cluster Setup (Part-1: Common Setup)

## Step 1: Hostname Setup (Optional but Recommended)

```bash
sudo hostnamectl set-hostname master-node

hostnamectl
```


## Step 2: Disable Swap

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled
free -h
```


## Step 3: Configure Kernel Modules and System Settings

```bash
# Make kernel modules persistent
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl settings
sudo sysctl --system
sudo sysctl -w net.ipv4.ip_forward=1
```

## Verify IP Forwarding

```bash
lsmod | grep br_netfilter
sysctl net.ipv4.ip_forward

# If the value is 1 for net.ipv4.ip_forward then OK
```



## Step 4: Install Container Runtime (Containerd)

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install containerd
sudo apt install -y containerd

# Create containerd configuration
sudo mkdir -p /etc/containerd
containerd config default \
| sed 's/SystemdCgroup = false/SystemdCgroup = true/' \
| sed 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' \
| sudo tee /etc/containerd/config.toml > /dev/null

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 5: Kubernetes Repository

```bash
# Install necessary packages
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Create GPG keyrings directory
sudo mkdir -p /etc/apt/keyrings

# Add Kubernetes GPG key and repository for v1.35
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list to recognize the new version
sudo apt update

# Verify kubernetes repo
cat /etc/apt/sources.list.d/kubernetes.list
```

## Step 6: Kubernetes Components Install

```bash
# Install kubeadm, kubelet, and kubectl

# Update package index and install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable kubelet
```

# Kubernetes 1.35 Cluster Setup (Part 2: Master Node Setup)

## Step 7: Initialize the Kubernetes Cluster

```bash
# Find your master node IP address
ip addr show

# Initialize the cluster (replace with your actual master IP)
sudo kubeadm init \
  --apiserver-advertise-address=<YOUR_MASTER_IP> \
  --pod-network-cidr=192.168.0.0/16
  ```
  **🔥 CRITICAL**: Save the `kubeadm join` command from the output!

  
## Step 8: Configure kubectl Access

```bash
# Setup kubectl configuration
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Cilium install (CNI Plugin)

```bash
# Cilium CLI version find out and download it.
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar -xzvf cilium-linux-${CLI_ARCH}.tar.gz -C /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Cilium version check
cilium version

# Cilium install with appropiate version.
cilium install --version v1.18.1 #choose from the last command (cilium version)_

# Status check
cilium status
```

## Step 10: Allow Scheduling on Master (Optional)

```bash
# Remove taint to allow pod scheduling on master
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Step 11: Join Worker Nodes

1. **Join the cluster** using the command from Step 7:

```bash
# Use the exact command from your master node initialization
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

## Verification

```bash
# Check all nodes are ready
kubectl get nodes

# Check all system pods are running
kubectl get pods --all-namespaces

# Test with sample application
kubectl create deployment test-nginx --image=nginx
kubectl get pods

# Clean up test
kubectl delete deployment test-nginx
```
