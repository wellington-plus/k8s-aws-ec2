# Kubernetes on EC2: Step-by-Step Guide

This guide will help you set up a 3-node Kubernetes (k8s) cluster on AWS EC2 instances. Each command is explained for clarity. **All commands must be run as root.**

---

## 1. Disable Swap (Required by Kubernetes)
```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
- `swapoff -a`: Disables swap immediately.
- The `sed` command comments out any swap entry in `/etc/fstab` to prevent swap from being enabled after reboot.

---

## 2. Enable Required Kernel Modules
```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```
- Creates a config file to load `overlay` and `br_netfilter` modules at boot.
- `modprobe` loads these modules now.

---

## 3. Set Kernel Parameters for Kubernetes Networking
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```
- Configures sysctl for network bridging and forwarding, required for Kubernetes networking.
- `sysctl --system` applies all sysctl configs.

---

## 4. Install Container Runtime (containerd)
```bash
apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update && apt-get install -y containerd.io

mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
```
- Installs dependencies and adds Docker's official GPG key and repo.
- Installs `containerd` (the container runtime).
- Generates default config for containerd and enables systemd cgroup driver (recommended for k8s).
- Restarts containerd to apply changes.

---

## 5. Install Kubernetes Components
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```
- Adds Kubernetes apt repo and installs `kubelet`, `kubeadm`, and `kubectl`.
- Holds their versions to prevent unintended upgrades.
- Enables and starts kubelet service.

---

## 6. Initialize the Control Plane (Run on Master Node Only)
```bash
kubeadm init

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```
- `kubeadm init` initializes the cluster.
- Sets up kubeconfig for kubectl access as the current user.

---

## 7. Verify Cluster Status
```bash
kubectl get nodes

# Wait until the cluster is ready
while [ $? -ne 0 ]; do
  sleep 10
  kubectl get nodes
  done
```
- Checks if the cluster is up and nodes are ready.
- The loop waits until `kubectl get nodes` succeeds.

---

## 8. Install a Pod Network Add-on (Weave)
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
- Installs Weave Net for pod networking. Required before scheduling pods.

---

**Repeat steps 1–5 on all worker nodes.**
- To join worker nodes, use the `kubeadm join ...` command output by `kubeadm init`.

---

**Your Kubernetes cluster is now ready!**

