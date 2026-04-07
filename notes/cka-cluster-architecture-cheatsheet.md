# CKA Cluster Architecture, Installation & Configuration Cheatsheet

> **Domain weight: 25%. Together with Troubleshooting (30%), these two domains = 55% of the exam.**

Covers: upgrades, RBAC, kubeconfig, etcd, certificates, Helm, Kustomize, CRDs, node management, kubeadm, CNI/CSI/CRI.

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

---

## Helm

### Repo Management
```bash
helm repo add <name> <url>              # add a chart repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                        # refresh repo indexes
helm repo list
helm search repo <keyword>              # search available charts
helm search repo nginx --versions       # list all versions
```

### Install
```bash
helm install <release> <chart>
helm install myapp bitnami/nginx
helm install myapp ./my-chart -n production --create-namespace
helm install myapp ./my-chart --set image.tag=2.0.0
helm install myapp ./my-chart -f custom-values.yaml
helm install myapp ./my-chart --dry-run --debug    # preview without applying
```

### Inspect Before Installing
```bash
helm show values bitnami/nginx          # see all configurable values
helm show chart bitnami/nginx           # show chart metadata
helm template myapp bitnami/nginx       # render manifests locally
```

### Upgrade
```bash
helm upgrade <release> <chart>
helm upgrade myapp bitnami/nginx --set replicaCount=3
helm upgrade myapp ./my-chart -f new-values.yaml
helm upgrade myapp ./my-chart --reuse-values    # keep existing values, add overrides
helm upgrade myapp ./my-chart --atomic          # rollback automatically on failure
```

### Rollback
```bash
helm rollback <release> [revision]
helm rollback myapp                     # roll back to previous revision
helm rollback myapp 2                   # roll back to specific revision number
helm history myapp                      # list all revisions
```

> **Gotcha:** `helm rollback` restores the **Kubernetes manifests** only. It does NOT restore database data or persistent volume contents.

### List & Uninstall
```bash
helm list -n <ns>                       # list releases in namespace
helm list -A                            # all namespaces
helm status myapp -n <ns>               # show release status
helm uninstall myapp -n <ns>
helm uninstall myapp --keep-history     # keep history for rollback
```

### Values Override Priority (high → low)
`--set` > `-f` files (rightmost wins) > chart defaults

---

## Kustomize

### Core Commands
```bash
kubectl apply -k <dir>                  # apply with kustomize
kubectl kustomize <dir>                 # preview output (doesn't apply)
kubectl diff -k <dir>                   # diff against cluster state
kubectl apply -k <dir> --dry-run=client # preview what would apply
```

### kustomization.yaml Structure
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production
namePrefix: prod-
nameSuffix: -v1

commonLabels:
  app: myapp
  env: prod

commonAnnotations:
  owner: team-a

resources:
- deployment.yaml
- service.yaml
- ../base              # reference a base layer

patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=debug
  - REPLICAS=3

secretGenerator:
- name: app-secret
  literals:
  - DB_PASS=supersecret
```

### Base/Overlay Pattern
```
project/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/kustomization.yaml     # resources: [../../base]
    └── prod/kustomization.yaml    # resources: [../../base] + patches
```

```bash
kubectl apply -k overlays/prod/
```

> **Gotcha:** `commonLabels` modifies selector labels too — don't add it to existing Deployments unless you want to break the selector.

> **Gotcha:** `configMapGenerator` and `secretGenerator` add a content hash suffix by default (e.g., `app-config-8dk7t`). Add `generatorOptions: {disableNameSuffixHash: true}` to suppress.

---

## CRDs and Custom Resources

### Check CRDs in the Cluster
```bash
kubectl get crd                                      # list all CRDs
kubectl get customresourcedefinitions               # same, long form
kubectl describe crd <name>                         # full CRD spec
kubectl get crd <name> -o yaml                      # export YAML
```

### Work with Custom Resources
```bash
# Once a CRD is installed, use kubectl like any native resource
kubectl get <plural-name>                           # e.g. kubectl get crontabs
kubectl get <plural-name> -n <ns>
kubectl describe <plural-name> <resource-name>
kubectl get <plural-name> -A
kubectl delete <plural-name> <resource-name>
```

> **Gotcha:** CRD objects themselves are **cluster-scoped**. Custom resources created from a CRD may be namespace-scoped or cluster-scoped depending on the CRD's `spec.scope`.

> **Gotcha:** Deleting a CRD **deletes all custom resource instances** of that type cluster-wide.

### What an Operator Does
An Operator = CRD + a controller (a pod that watches the CRD and reconciles state). In the exam you're unlikely to write an Operator, but you may need to:
```bash
kubectl get crd                              # see what CRDs the operator installed
kubectl get <custom-resource-type> -A       # see all instances
kubectl describe <custom-resource> <name>   # check status conditions
```

---

## Node Management

### Cordon (mark unschedulable, existing pods stay)
```bash
kubectl cordon <node>
kubectl get nodes   # shows SchedulingDisabled
```

### Drain (evict pods + cordon)
```bash
# Standard drain — works for most exam tasks
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# With grace period
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=60

# Force delete pods not managed by a controller
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force

# Dry run first
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --dry-run=client
```

> **Gotcha:** Drain will **hang** if pods exist that aren't managed by a controller AND you didn't pass `--force`. It will also hang if pods have `emptyDir` and you didn't pass `--delete-emptydir-data`.

### Uncordon (re-enable scheduling)
```bash
kubectl uncordon <node>
kubectl get nodes   # Back to Ready
```

### Typical Maintenance Workflow
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# ... perform maintenance (upgrade kubelet, reboot, etc.) ...
kubectl uncordon <node>
kubectl get nodes   # verify Ready
```

---

## kubeadm Init & Join

### Initialize Control Plane
```bash
# For Flannel CNI
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# For Calico CNI
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Specify API server IP
sudo kubeadm init \
  --apiserver-advertise-address=<control-plane-ip> \
  --pod-network-cidr=10.244.0.0/16

# Dry run (validate without applying)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --dry-run
```

### Post-Init Setup
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes        # control plane should show NotReady (no CNI yet)
```

### Join Worker Nodes
```bash
# On control plane: generate a fresh join command
kubeadm token create --print-join-command

# On worker node: run the output from above
sudo kubeadm join <cp-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> **Gotcha:** Default token TTL is **24 hours**. If the token expired, run `kubeadm token create --print-join-command` to generate a fresh one.

### Regenerate Static Pod Manifests
```bash
# If a manifest is missing or corrupt, regenerate it without reinitializing:
sudo kubeadm init phase control-plane apiserver
sudo kubeadm init phase control-plane controller-manager
sudo kubeadm init phase control-plane scheduler
sudo kubeadm init phase control-plane all     # all three
sudo kubeadm init phase etcd local            # etcd only
```

---

## CNI / CSI / CRI

### CNI (Container Network Interface) — Pod Networking
```bash
# Check which CNI is installed
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium|weave'

# Check CNI config on node
ls /etc/cni/net.d/          # CNI config files
ls /opt/cni/bin/            # CNI plugin binaries

# No CNI = pods stay Pending / NetworkUnavailable: True on node
kubectl describe node <node> | grep NetworkUnavailable
```

### CSI (Container Storage Interface) — Persistent Storage
```bash
# Check installed CSI drivers
kubectl get csidrivers

# Check storage classes (provisioner field shows which CSI driver)
kubectl get sc

# CSI driver pod missing = PVCs will stay Pending
kubectl get pods -A | grep csi
```

### CRI (Container Runtime Interface) — Container Runtime
```bash
# Check which runtime each node uses
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'

# Check CRI socket (on node)
ls /var/run/containerd/containerd.sock     # containerd (most common)
ls /var/run/crio/crio.sock                 # CRI-O

# Use crictl (replaces docker CLI)
sudo crictl ps -a                          # list containers
sudo crictl logs <container-id>           # container logs
sudo crictl inspect <container-id>        # container config
```

> **Gotcha:** `docker` CLI is no longer available in modern Kubernetes clusters. Use `crictl` for node-level container inspection.

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
| `helm rollback` restoring data | Rollback only restores manifests, NOT PV data |
| `kubectl apply -k .` vs `kubectl kustomize .` | `apply -k` deploys; `kustomize .` only renders output |
| kubeadm join token expired | Run `kubeadm token create --print-join-command` to get a fresh token |
| Deleting a CRD | Also deletes ALL custom resources of that type — irreversible |
| Forgetting `--ignore-daemonsets` on drain | Drain hangs waiting for DaemonSet pods that can't be evicted |

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
