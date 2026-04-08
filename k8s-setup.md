```
#kubernetes 1.35 cluster setup (part-1: common setup)

##Step 1: Hostname setup (Optional but Recommended)
```
sudo hostnamectl set-hostname master-node

hostnamectl```


##Step 2: Disable swap
```
# Disable swap immediately

sudo swapoff -a

# Disable swap permanently

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# verify swap is disabled

free -h```


##Step 3: Configure Kernel Modules and System Settings
```
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

sudo sysctl -w net.ipv4.ip_forward=1```

##Verify IP forwarding:
```
lsmod | grep br_netfilter
sysctl net.ipv4.ip_forward

if the value 1 of net.ipv4.ip_forward then oky```



##Step 4: Install Container Runtime (Containerd)

```
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
sudo systemctl enable containerd ```

##Step 5: Kubernetes Repository

```
#1.Update Repository and Install Dependencies

# Install necessary packages
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Create GPG keyrings directory
sudo mkdir -p /etc/apt/keyrings

#2. Add Kubernetes v1.35 GPG Key and Repository

# Add Kubernetes GPG key and repository for v1.35
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list to recognize the new version
sudo apt update


#Now verification of kubernetes repo

cat /etc/apt/sources.list.d/kubernetes.list```

##Step 6: Kubernetes Components Install

```
# Install kubeadm, kubelet, and kubectl

# Update package index and install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable kubelet

```



##Kubernetes 1.35 Cluster Setup (Part 2: Master Node Setup)




