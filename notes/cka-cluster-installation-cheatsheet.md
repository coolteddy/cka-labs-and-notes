# CKA Cluster Installation Cheatsheet

Kubernetes two-node cluster using `kubeadm` on Multipass VMs (Ubuntu 22.04).

---

## Architecture

```
Host (Ubuntu — Dell XPS 13)
  ├── k8s-master  (2 CPU, 2GB RAM, 20GB disk) — control plane
  └── k8s-worker  (2 CPU, 2GB RAM, 20GB disk) — worker node
```

- **Kubernetes version:** v1.32.x
- **CNI:** Calico v3.29.0
- **Pod CIDR:** 192.168.0.0/16
- **Container runtime:** containerd

---

## STEP 1 — Create Multipass VMs

```bash
# Install multipass
sudo snap install multipass

# Create VMs (use 22.04 — more stable with Calico than 24.04)
multipass launch --name k8s-master --cpus 2 --memory 2G --disk 20G 22.04
multipass launch --name k8s-worker --cpus 2 --memory 2G --disk 20G 22.04

# Verify
multipass list

# Shell into VMs
multipass shell k8s-master
multipass shell k8s-worker
```

---

## STEP 2 — Common Setup (Run on BOTH nodes)

```bash
# Disable swap (kubelet requires swap off)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl networking params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get update -y
sudo apt-get install -y containerd

# Configure containerd with systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions to prevent accidental upgrades
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

---

## STEP 3 — Initialize Cluster (Master ONLY)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Set up kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify control plane
kubectl get nodes
kubectl get pods -n kube-system
```

---

## STEP 4 — Install Calico CNI (Master ONLY)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml

# Watch pods come up
kubectl get pods -n kube-system -w
```

---

## STEP 5 — Join Worker Node

```bash
# On master — generate join command
kubeadm token create --print-join-command

# On worker — run the output from above with sudo
sudo kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## STEP 6 — Verify Cluster

```bash
# Both nodes should show Ready
kubectl get nodes -o wide

# All pods should show Running
kubectl get pods -n kube-system

# Test DNS
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes
# Expected: Name: kubernetes.default.svc.cluster.local → Address: 10.96.0.1
```

---

## STEP 7 — Access Cluster from Host Machine

```bash
# On host (not inside VM)
mkdir -p $HOME/.kube
multipass exec k8s-master -- sudo cat /etc/kubernetes/admin.conf > $HOME/.kube/config

# Verify
kubectl get nodes
```

---

## Multipass Quick Reference

```bash
multipass list                              # list VMs
multipass shell k8s-master                  # shell into VM
multipass stop k8s-master k8s-worker        # stop VMs
multipass start k8s-master k8s-worker       # start VMs
multipass delete k8s-master k8s-worker && multipass purge  # delete all
```

---

## Troubleshooting Notes

| Issue | Cause | Fix |
|-------|-------|-----|
| Calico-node CrashLoopBackOff (exit 0) on worker | Ubuntu 24.04 instability with Calico | Use Ubuntu 22.04 |
| ImagePullBackOff | Intermittent internet | Wait or `sudo crictl pull <image>` |
| kubectl on worker: connection refused | Worker has no kubeconfig | Run kubectl from master only |
| DNS NXDOMAIN for some entries | busybox searches multiple domains | Normal — check for `kubernetes.default.svc.cluster.local` |
| Join command token expired | Tokens expire after 24h | `kubeadm token create --print-join-command` |

---

## Key Concepts (Why We Do Each Step)

| Step | Why |
|------|-----|
| `swapoff` | kubelet refuses to start with swap enabled |
| `overlay` module | OverlayFS — container image layering |
| `br_netfilter` module | Allows iptables to see bridged pod traffic |
| `SystemdCgroup = true` | Containerd and kubelet must use same cgroup driver |
| `apt-mark hold` | Prevents accidental k8s version upgrades |
| `--pod-network-cidr` | Reserves IP range for pods — must match CNI config |
