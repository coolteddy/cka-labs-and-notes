# CKA Cluster Architecture, Installation & Configuration Cheatsheet

Covers the 25% domain: upgrades, RBAC, kubeconfig, etcd backup/restore, certificate management.

---

## Cluster Upgrade (kubeadm)

### Key Rule
- Always upgrade control plane first, then workers
- Can only upgrade one minor version at a time (e.g. 1.32 → 1.33, not 1.32 → 1.34)
- `kubeadm upgrade apply` — control plane only
- `kubeadm upgrade node` — workers only (common exam mistake)

### Control Plane Upgrade

```bash
# 1. Update apt repo to new version
sudo sed -i 's|/v1.32/|/v1.33/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.33.0-1.1
sudo apt-mark hold kubeadm

# 3. Verify upgrade plan
sudo kubeadm upgrade plan

# 4. Apply upgrade
sudo kubeadm upgrade apply v1.33.0

# 5. Drain node
kubectl drain k8s-master --ignore-daemonsets --delete-emptydir-data

# 6. Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 7. Uncordon
kubectl uncordon k8s-master
```

### Worker Node Upgrade

```bash
# SSH into worker node first
# 1. Update apt repo and upgrade kubeadm (same as above)

# 2. Upgrade node config (NOT upgrade apply)
sudo kubeadm upgrade node

# 3. Drain from master
kubectl drain k8s-worker --ignore-daemonsets --delete-emptydir-data

# 4. Upgrade kubelet and kubectl on worker
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.0-1.1 kubectl=1.33.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. Uncordon from master
kubectl uncordon k8s-worker
```

### Verify
```bash
kubectl get nodes
```

---

## RBAC

### Roles and Bindings

```bash
# Create Role (namespace-scoped)
kubectl create role pod-reader -n default \
  --verb=get,list,watch \
  --resource=pods

# Create RoleBinding
kubectl create rolebinding alice-pod-reader -n default \
  --role=pod-reader \
  --user=alice

# Create ClusterRole (cluster-wide)
kubectl create clusterrole deployment-manager \
  --verb=get,list,create,update,delete \
  --resource=deployments

# Create ClusterRoleBinding
kubectl create clusterrolebinding deployer-binding \
  --clusterrole=deployment-manager \
  --serviceaccount=default:deployer
```

### ServiceAccount

```bash
# Create ServiceAccount
kubectl create sa secret-reader -n default

# Bind role to ServiceAccount
kubectl create rolebinding secret-read-bind -n default \
  --role=secret-read \
  --serviceaccount=default:secret-reader

# Create pod using ServiceAccount
kubectl run app-pod --image=nginx --dry-run=client -o yaml > pod.yaml
# Add to pod spec: serviceAccountName: secret-reader
kubectl apply -f pod.yaml
```

### Verify Permissions

```bash
# For a user
kubectl auth can-i list pods --as=alice -n default

# For a ServiceAccount
kubectl auth can-i list pods --as=system:serviceaccount:default:secret-reader -n default
```

### Pod YAML with ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: secret-reader
  containers:
  - name: app-pod
    image: nginx
```

---

## Kubeconfig

### Add new cluster/user/context

```bash
# Add cluster
kubectl config set-cluster my-cluster \
  --server=https://10.124.224.212:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true

# Add user (with cert files)
kubectl config set-credentials alice \
  --client-certificate=/home/ubuntu/alice.crt \
  --client-key=/home/ubuntu/alice.key \
  --embed-certs=true

# Add context
kubectl config set-context alice-context \
  --cluster=my-cluster \
  --user=alice

# Switch context
kubectl config use-context alice-context

# Set default namespace for current context
kubectl config set-context --current --namespace=default
```

### Useful commands

```bash
kubectl config view                          # view full kubeconfig
kubectl config view --minify                 # view current context only
kubectl config view --minify | grep namespace
kubectl config current-context               # show current context
kubectl config get-contexts                  # list all contexts
kubectl config use-context <name>            # switch context
```

---

## Certificate Signing Request (CSR) — Create a User

```bash
# 1. Generate private key
openssl genrsa -out alice.key 2048

# 2. Generate CSR file
openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=dev"

# 3. Create Kubernetes CSR object
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice
spec:
  request: $(cat alice.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 4. Approve
kubectl certificate approve alice

# 5. Extract certificate
kubectl get csr alice -o jsonpath='{.status.certificate}' | base64 -d > alice.crt

# 6. Verify
ls -la alice.key alice.crt
```

---

## Certificate Expiry & Renewal

```bash
# Check expiry of all certs
sudo kubeadm certs check-expiration

# Renew all certs
sudo kubeadm certs renew all

# Renew specific cert
sudo kubeadm certs renew apiserver

# Restart control plane to pick up new certs
sudo systemctl restart kubelet
```

---

## etcd Backup & Restore

### Backup

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

# Verify
sudo ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db
```

### Restore

```bash
# 1. Restore snapshot to new directory
sudo etcdutl --data-dir /var/lib/etcd-restore snapshot restore /opt/etcd-backup.db

# 2. Update etcd static pod manifest — change ALL THREE:
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

Change these three lines:
```yaml
# In containers.command:
- --data-dir=/var/lib/etcd-restore        # was /var/lib/etcd

# In volumeMounts:
- mountPath: /var/lib/etcd-restore        # was /var/lib/etcd

# In volumes.hostPath:
path: /var/lib/etcd-restore               # was /var/lib/etcd
```

```bash
# 3. Restart kubelet to reconcile static pods
sudo systemctl restart kubelet

# 4. Verify
kubectl get pods -n kube-system
kubectl get nodes
```

---

## Exam Gotchas

| Mistake | Correct Approach |
|---------|-----------------|
| Running `kubeadm upgrade apply` on worker | Workers use `kubeadm upgrade node` only |
| Forgetting `-n <namespace>` in RBAC commands | Always specify namespace explicitly |
| `--as=alice` for ServiceAccount | Use `--as=system:serviceaccount:<ns>:<name>` |
| Changing only hostPath in etcd restore | Change `--data-dir`, `mountPath`, AND `hostPath.path` |
| Forgetting to restart after cert renewal | Run `sudo systemctl restart kubelet` |
| Using `kubectl config use-context kubernetes` | Use full name e.g. `kubernetes-admin@kubernetes` |
| Forgetting `--embed-certs=true` in kubeconfig | Without it, cert file path must always exist |
| Using apt etcdctl (v3.3) with v3.5 flags | Use etcdctl from etcd release tarball |

---

## Key File Locations

| File | Path |
|------|------|
| Cluster CA cert | `/etc/kubernetes/pki/ca.crt` |
| etcd CA cert | `/etc/kubernetes/pki/etcd/ca.crt` |
| etcd server cert | `/etc/kubernetes/pki/etcd/server.crt` |
| etcd server key | `/etc/kubernetes/pki/etcd/server.key` |
| Static pod manifests | `/etc/kubernetes/manifests/` |
| kubeconfig (admin) | `/etc/kubernetes/admin.conf` |
| kubeconfig (user) | `~/.kube/config` |
| etcd data dir | `/var/lib/etcd` |
| kubelet config | `/var/lib/kubelet/config.yaml` |
