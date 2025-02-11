# rocky-k8s-cluster
Installing a Kubernetes cluster on RHEL 8 involves preparing nodes, configuring container runtimes, and initializing the control plane. Here's a consolidated guide based on best practices from multiple sources:

## Prerequisites
- **Nodes**: At least two RHEL 8 systems (1 master, 1+ workers) with static IPs
- **Resources**: 2+ GB RAM and 2+ CPUs per node
- **Network**: Disable firewalls or allow Kubernetes ports (6443, 10250, etc.)

---

## 1. System Preparation (Both Nodes)
**Disable Swap**  
Kubernetes requires swap to be disabled for stability:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  # Permanent disable [3][8]
```

**Configure SELinux**  
Set SELinux to permissive mode:
```bash
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo setenforce 0  # Immediate effect [3][4]
```

**Network Configuration**  
Enable kernel modules and IP forwarding:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system  # Apply changes [3][4]
```

---

## 2. Container Runtime Installation
**Option A: CRI-O (Recommended)**  
For Kubernetes 1.23+:
```bash
export CRIO_VERSION=v1.30
cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y cri-o
sudo systemctl enable --now crio [3][4]
```

**Option B: Docker**  
For legacy support:
```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
```

---

## 3. Kubernetes Components Installation
Add repository and install tools:
```bash
KUBERNETES_VERSION=v1.30
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet [2][3][8]
```

---

## 4. Master Node Initialization
```bash
sudo kubeadm init --pod-network-cidr=192.168.10.0/16  # Or 10.244.0.0/16 for Flannel
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config [3][6][8]
```

**Install Network Plugin**  
For Calico:
```bash
kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calico.yaml
```
For Flannel:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml [4][8]
```

---

## 5. Worker Node Joining
Use the join command generated during master initialization:
```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> [3][4]
```

---

## Verification
**Check Cluster Status**  
On master node:
```bash
kubectl get nodes  # Should show all nodes as 'Ready' [3][6]
```

**Test Deployment**  
Create sample Nginx deployment:
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort [6]
```

---

**Key Considerations**:
- Use consistent Kubernetes versions across all nodes[6]
- Ensure time synchronization (chrony/NTP) between nodes
- For production, consider etcd backups and high availability[7]
- Recent versions may require containerd instead of Docker[9]
