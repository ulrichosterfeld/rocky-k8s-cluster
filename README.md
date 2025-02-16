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

**Configure networking in master and worker node**
Some additional network configuration is required for your master and worker nodes to communicate effectively. 
On each node, edit the /etc/hosts file and add the following lines:
```bash
sudo vi /etc/hosts
192.168.178.20 k8s-cp-01
192.168.178.21 k8s-wrk-01
```
Save and exit the configuration file.
Next, install the traffic control utility package:
```bash
sudo dnf install -y iproute-tc
```
**Allow firewall rules for k8s**
For seamless communication between the Master and worker node, you need to configure the firewall and allow some pertinent ports and services as outlined below.
On Master node, allow following ports:
```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --reload
```
On Worker node, allow following ports:
```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp                                                 
sudo firewall-cmd --reload
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
To install CRI-O, set the $CRIO_VERSION environment variable to match your CRI-O version. 
For instance, to install CRI-O version 1.30 set the $CRIO_VERSION as shown:

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

sudo systemctl enable crio
sudo systemctl start crio
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

sudo dnf install kubelet kubeadm kubectl -y

sudo systemctl enable kubelet
sudo systemctl start kubelet

```

---

## 4. Master Node Initialization
```bash
sudo kubeadm init --pod-network-cidr=192.168.10.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config [3][6][8]
```
At the very end of the output, you will be given the command to run on worker nodes to join the cluster. We will come to that later in the next step.
Also, be sure to remove taints from the master node:
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

**Install Network Plugin**  
After Installing Calico CNI, nodes state will change to Ready state, DNS service inside the cluster would be functional and containers can start communicating with each other.
Calico provides scalability, high performance, and interoperability with existing Kubernetes workloads. It can be deployed on-premises and on popular cloud technologies such as Google Cloud, AWS and Azure.
To install Calico CNI, run the following command from the master node:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```
To confirm if the pods have started, run the command:
```bash
kubectl get pods -n kube-system
```
You should see that each pod is ‘READY’ and has the ‘RUNNING’ status as shown in the third column.
To verify the master node’s availability in the cluster, run the command:
```bash
kubectl get nodes
NAME           STATUS   ROLES            AGE    VERSION
k8s-cp-01      Ready    control-plane    9m56s  v1.30.4
```
In addition, you can retrieve more information using the -o wide options:
```bash
kubectl get nodes -o wide
```
The above output confirms that the master node is ready. 
Additionally, you can check the pod namespaces:
```bash
kubectl get pods --all-namespaces
```
---

## 5. Worker Node Joining
Use the join command generated during master initialization:
```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> 
```

---

## Verification
**Check Cluster Status**  
On master node:
```bash
kubectl get nodes  # Should show all nodes as 'Ready' 
```

**Test Deployment**  
Create sample Nginx deployment:
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
```

---

**Key Considerations**:
- Use consistent Kubernetes versions across all nodes
- Ensure time synchronization (chrony/NTP) between nodes
- For production, consider etcd backups and high availability
