# CKA Cheatsheet: Services & Networking, Workloads & Scheduling, Storage

Covers domains: Services & Networking (20%), Workloads & Scheduling (15%), Storage (10%)

---

## Quick Setup

```bash
# vim settings — add to ~/.vimrc
set ts=2 sw=2 et

# Namespace shortcut
kns() { kubectl config set-context --current --namespace="$1"; }

# Check current namespace
kubectl config view --minify | grep namespace
```

---

## Services & Networking

### Service Types

```bash
# ClusterIP (default) — internal only
kubectl expose deployment api -n api-prod \
  --name=api-svc --port=80 --target-port=8080 --type=ClusterIP

# NodePort — external via node IP
kubectl expose deployment web --name=web-svc --port=80 \
  --target-port=80 --type=NodePort

# Check NodePort assigned
kubectl get svc web-svc -n <ns>
# Access: <node-ip>:<nodePort>
kubectl get nodes -o wide   # get node IPs
```

### Diagnose Broken Service

```bash
# 1. Check endpoints — empty = selector mismatch
kubectl get endpoints <svc> -n <ns>

# 2. Check pod labels vs service selector
kubectl get pods -n <ns> --show-labels
kubectl describe svc <svc> -n <ns> | grep Selector

# 3. Fix selector or targetPort
kubectl edit svc <svc> -n <ns>
```

**Failure hints:**
- `connection refused` → wrong `targetPort` or app not listening on that port
- `timeout` → NetworkPolicy blocking, wrong selector, or no endpoints

### DNS

```bash
# FQDN format
<service>.<namespace>.svc.cluster.local

# Test from temporary pod
kubectl run tmp --image=busybox --restart=Never --rm -i -- \
  nslookup api-svc.api-prod.svc.cluster.local

# Test HTTP
kubectl run tmp --image=busybox --restart=Never --rm -i -- \
  wget -qO- --timeout=5 http://api-svc.api-prod.svc.cluster.local
```

### CoreDNS Troubleshooting

```bash
# Test DNS resolution
kubectl run debug --image=busybox --restart=Never --rm -i -- nslookup google.com

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Fix broken upstream — edit ConfigMap
kubectl edit configmap coredns -n kube-system
# Change: forward . 192.0.2.1  →  forward . /etc/resolv.conf

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

**Note:** `forward . /etc/resolv.conf` uses the node's DNS. CoreDNS pods inherit the node's resolv.conf via `dnsPolicy: Default`.

### Ingress

```bash
# Create ingress imperatively
kubectl create ingress app-ingress -n <ns> \
  --class=nginx \
  --rule="app.example.com/v1=svc-v1:80" \
  --rule="app.example.com/v2=svc-v2:80"

# With TLS
kubectl create ingress webapp-ingress -n <ns> \
  --class=nginx \
  --rule="secure.example.com/=webapp-svc:80,tls=webapp-tls"

# Lab only — on exam, cert/key files are pre-provided
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.example.com"

# Create TLS secret
kubectl create secret tls webapp-tls -n <ns> \
  --cert=tls.crt --key=tls.key

# Test with Host header (no real DNS)
curl -k -H "Host: secure.example.com" https://<node-ip>:30443
curl -H "Host: app.example.com" http://<node-ip>:30080/v1
```

**Ingress YAML key sections:**
```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: webapp-tls    # must match secret name
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

**Gotcha:** `tls.hosts[]` and `rules[].host` must match exactly.

**Rewrite annotation** (when backend only serves `/`):
```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

### NetworkPolicy

#### Default Deny Patterns

```yaml
# Deny all ingress only
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # NO ingress field = deny all ingress

# Deny all ingress AND egress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Trap:** `ingress: [{}]` = allow ALL ingress. `ingress: []` or omitting = deny all.

#### Allow Traffic Between Pods

```yaml
# Policy 1: Allow egress FROM frontend TO backend port 80
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 80

---
# Policy 2: Allow ingress TO backend FROM frontend port 80
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
```

**Key concept:** NetworkPolicy is **stateful** (conntrack). Only model the direction of initiation — return traffic is automatic. Two policies needed: egress on source pod + ingress on destination pod.

#### Cross-Namespace Traffic

```yaml
# Allow from specific namespace (uses built-in label)
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring

# Allow from outside a namespace (NotIn)
ingress:
- from:
  - namespaceSelector:
      matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: NotIn
        values:
        - my-namespace
```

**Note:** `kubernetes.io/metadata.name` is auto-applied to all namespaces since k8s 1.21.

#### Test NetworkPolicy

```bash
# Positive test (should succeed)
kubectl run test --image=busybox --restart=Never --rm -i \
  --labels="app=frontend" -n prod \
  -- wget -qO- --timeout=5 http://<backend-ip>

# Negative test (should fail/timeout)
kubectl run stranger --image=busybox --restart=Never --rm -i -n default \
  -- wget -qO- --timeout=5 http://<backend-ip>

# TCP port test (non-HTTP)
kubectl run test --image=busybox --restart=Never --rm -i \
  --labels="app=api" -n prod \
  -- nc -zv <db-ip> 5432
```

Always test **both positive and negative** cases.

---

## Workloads & Scheduling

### Deployment Operations

```bash
# Create
kubectl create deploy web --image=nginx:1.25 --replicas=3 -n <ns>

# Scale
kubectl scale deploy web --replicas=5 -n <ns>

# Update image
kubectl set image deployment/web nginx=nginx:1.26 -n <ns>

# Update resources
kubectl set resources deployment/web -n <ns> \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi

# Update env from ConfigMap/Secret (Deployments only, NOT bare pods)
kubectl set env deployment/web --from=configmap/app-config -n <ns>
kubectl set env deployment/web --from=secret/app-secret -n <ns>

# Rollout
kubectl rollout status deployment/web -n <ns>
kubectl rollout history deployment/web -n <ns>
kubectl rollout undo deployment/web -n <ns>
kubectl rollout undo deployment/web --to-revision=1 -n <ns>
```

**Note:** `kubectl set env` works on Deployments (triggers rolling update). Does NOT work on bare Pods — pods are immutable after creation.

### Multi-Container Pod

```bash
# Generate base YAML, then edit
kubectl run app-pod --image=nginx:1.25 --dry-run=client -o yaml > pod.yaml
# Edit: add second container, envFrom, volumeMounts
kubectl apply -f pod.yaml
```

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.25
    envFrom:
    - configMapRef:
        name: app-config      # inject all keys from ConfigMap
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD    # inject single key from Secret
  - name: log-shipper         # sidecar container
    image: busybox
    command: ["sh", "-c", "while true; do echo logging; sleep 5; done"]
```

### Scheduling — nodeSelector vs nodeAffinity

| | nodeSelector | nodeAffinity |
|---|---|---|
| Requirement type | Hard only | Hard (`required`) or Soft (`preferred`) |
| Operators | `=` only | `In`, `NotIn`, `Exists`, `Gt`, `Lt` |
| Use when | Simple key=value | Task says "preferred" or needs operators |

**Important:** Taint ≠ Label. They are separate:
- **Label** → attracts pods (nodeSelector/nodeAffinity)
- **Taint** → repels pods (unless toleration exists)

Pod needs **both** label match AND taint toleration to land on a node.

```yaml
spec:
  nodeSelector:
    disktype: ssd              # simple — hard requirement

  # OR use nodeAffinity for soft/complex requirements:
  affinity:
    nodeAffinity:
      # Hard requirement
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

      # Soft requirement (preferred)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1              # required, 1-100
        preference:            # single object, NOT a list
          matchExpressions:    # IS a list
          - key: zone
            operator: In
            values:
            - us-east

  tolerations:
  - key: env
    operator: Equal
    value: prod
    effect: NoSchedule
```

**Tip:** Use `kubectl explain` to check structure. `[]` in output = list, no `[]` = single object.

### HPA

```bash
# Create HPA imperatively
kubectl autoscale deployment web -n <ns> \
  --name=web-hpa \
  --min=2 --max=8 --cpu-percent=60

# Check
kubectl get hpa web-hpa -n <ns>
```

**Note:** Flag is `--cpu-percent`, not `--cpu`.

### PodDisruptionBudget

```bash
kubectl create poddisruptionbudget critical-pdb -n <ns> \
  --selector=app=critical-app \
  --min-available=3
# Also supports: --max-unavailable=1
```

### PriorityClass

```bash
kubectl create priorityclass high-priority --value=1000
# --global-default omitted = false (default)
# Only ONE PriorityClass can have globalDefault: true
```

### Node Drain & Uncordon

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
# --ignore-daemonsets: skip DaemonSet pods
# --delete-emptydir-data: evict pods using emptyDir volumes
# --force: evict bare pods (not managed by controller)

kubectl uncordon <node>
```

### Resource Requests & Limits

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**Units:** `m` = millicores, `Mi` = mebibytes (binary), `M` = megabytes (SI). Use `Mi`/`Gi` unless task specifies otherwise.

### StatefulSet

Key differences from Deployment:
- Pods get stable ordered names: `mysql-0`, `mysql-1`
- Each pod gets its own PVC via `volumeClaimTemplates`
- PVC naming: `<template-name>-<pod-name>`
- Requires a headless service (`clusterIP: None`)
- PVCs are **not deleted** when StatefulSet is deleted

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  serviceName: mysql-headless    # required — headless service name
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:          # auto-creates PVC per pod
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: local-path
      resources:
        requests:
          storage: 1Gi
```

### Helm

```bash
# Add repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Find chart
helm search repo nginx
helm search repo nginx --versions   # show all versions

# Install
helm install web-release bitnami/nginx -n <ns> \
  --set replicaCount=2

# Find values to override
helm show values bitnami/nginx | grep -i replica

# Check installed release
helm list -n <ns>
helm get metadata web-release -n <ns>   # shows chart name + version

# Upgrade
helm upgrade web-release bitnami/nginx -n <ns> --set replicaCount=3

# Rollback (creates NEW revision)
helm history web-release -n <ns>
helm rollback web-release 1 -n <ns>

# Uninstall
helm uninstall web-release -n <ns>

# Inspect deployed manifests
helm get manifest web-release -n <ns>

# Preview without installing
helm template web-release bitnami/nginx --set replicaCount=2
```

---

## Storage

### PV + PVC Binding Rules

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data

---
# PVC — must match PV exactly
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: <ns>
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual   # must match PV storageClassName
```

**Binding requirements:**
- `capacity` must match
- `storageClassName` must match exactly
- `accessModes` must be compatible

**Trap:** `storageClassName: ""` ≠ `storageClassName: manual`. Empty string only matches PVs with explicitly empty storageClassName.

**Diagnose PVC stuck Pending:**
```bash
kubectl describe pvc <name> -n <ns>
# Check: Events — usually "no persistent volumes available" or "storageClass mismatch"
kubectl get pv   # check PV status and storageClassName
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-sc
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer   # bind only when pod is scheduled
allowVolumeExpansion: true
```

**`WaitForFirstConsumer`** — PVC stays `Pending` until a pod using it is created. Expected behaviour.

### Expand PVC

```bash
# StorageClass must have allowVolumeExpansion: true
kubectl edit pvc <name> -n <ns>
# Change: storage: 1Gi → storage: 2Gi

# May need pod restart for filesystem resize to take effect
```

### Mount PVC in Pod

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc   # typo here causes pod to stay Pending
```

**Verify mount:**
```bash
kubectl exec -n <ns> <pod> -- df -h /data
```

### Best Practices

- Always `kubectl get <resource> -o yaml > file.yaml` before deleting
- PVC `storageClassName` is immutable — must delete and recreate if wrong
- Check endpoints are populated before debugging the pod
- PVCs from StatefulSet `volumeClaimTemplates` are NOT deleted with the StatefulSet

---

## Diagnostic Toolkit

### General Pod Issues

```bash
# Pod not scheduling
kubectl describe pod <pod> -n <ns>
# Check: Events — "didn't match node selector", "untolerated taint", "insufficient cpu"

# Pod not ready — check probe failures
kubectl describe pod <pod> -n <ns>
# Check: Liveness/Readiness probe failures in Events
# Fix: kubectl edit deployment <name> -n <ns> → change probe path (e.g. /healthz → /)

# CrashLoopBackOff
kubectl logs <pod> -n <ns> --previous
```

### Quick Connectivity Tests

```bash
# HTTP
kubectl run tmp --image=busybox --restart=Never --rm -i -- \
  wget -qO- --timeout=5 http://<target>

# HTTPS
kubectl run tmp --image=curlimages/curl --restart=Never --rm -i -- \
  curl -sk --max-time 5 https://<target>

# TCP port
kubectl run tmp --image=busybox --restart=Never --rm -i -- \
  nc -zv <host> <port>
```

**Timeout values — no unit suffix:**
- `wget --timeout=5` (plain number, seconds)
- `curl --max-time 5` (plain number, seconds)
- `wget --timeout=5s` → **WRONG**, causes error

### RBAC Verify

```bash
kubectl auth can-i list pods -n <ns> --as=system:serviceaccount:<ns>:<sa-name>
kubectl auth can-i create deployments --as=alice
```

### ResourceQuota

```bash
kubectl create quota my-quota -n <ns> \
  --hard=pods=10,cpu=2,memory=1Gi

kubectl describe resourcequota my-quota -n <ns>
```

---

## Exam Gotchas

| Trap | Correct Approach |
|------|-----------------|
| `ingress: [{}]` | Empty object `{}` = allow all. Omit `ingress:` for deny-all |
| NetworkPolicy bidirectional | Stateful — only model initiation direction. Need egress on source + ingress on dest |
| `storageClassName: ""` | Does NOT match `storageClassName: manual`. Must match exactly |
| `kubectl set env` on bare pod | Pods are immutable. Only works on Deployments. Use `--dry-run -o yaml` for pods |
| `--cpu-percent` flag for HPA | Not `--cpu`. Use `kubectl autoscale --help` if unsure |
| `wget --timeout=5s` | Invalid. Use `--timeout=5` (plain number) |
| nodeSelector vs nodeAffinity | nodeSelector = hard only. nodeAffinity = hard + preferred + operators |
| Taint ≠ Label | Taint repels, label attracts. Pod needs both toleration AND label match |
| `helm rollback` revision | Creates a NEW revision. History goes 1→2→3, not back to 1 |
| PVC storageClassName | Immutable. Delete and recreate if wrong. Always save YAML first |
| `preference:` in nodeAffinity | Single object, NOT a list. `matchExpressions` under it IS a list |
| Ingress TLS secret name | Must match exactly in both `tls.secretName` and the secret resource |
| `1G` vs `1Gi` | Different values. Match exactly what the task specifies |
| `kubectl get endpoints svc <name>` | `svc` treated as resource name. Use `kubectl get endpoints <name> -n <ns>` |

---

## kubectl set — Quick Reference

```bash
kubectl set image deployment/<name> <container>=<image>:<tag>
kubectl set resources deployment/<name> --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=256Mi
kubectl set env deployment/<name> --from=configmap/<name>
kubectl set env deployment/<name> --from=secret/<name>
kubectl set env deployment/<name> KEY=value
kubectl set env deployment/<name> KEY-              # remove env var
kubectl set serviceaccount deployment/<name> <sa-name>
```

---

## vim Tips

```bash
# Add to ~/.vimrc (one line)
set ts=2 sw=2 et

# In vi — paste without auto-indent mangling
:set paste
# paste your YAML
:set nopaste

# Generate base YAML, copy for variations
kubectl run pod1 --image=nginx --dry-run=client -o yaml > pod1.yaml
cp pod1.yaml pod2.yaml
vi pod2.yaml
```
