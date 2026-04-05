# CKA Cheatsheet: Services & Networking, Workloads & Scheduling, Storage, Cluster Admin

Covers domains: Services & Networking (20%), Workloads & Scheduling (15%), Storage (10%), Cluster Architecture (25%)

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

**Always specify `--class=nginx`** when creating ingress — exam clusters use nginx ingress controller:
```bash
kubectl create ingress echo -n echo-sound \
  --rule="example.org/echo=echoserver-service:8080" \
  --class=nginx
```

**pathType: Prefix vs Exact:**
- `Exact` → matches only `/echo` literally (nginx may normalise path and fail)
- `Prefix` → matches `/echo`, `/echo/`, `/echo/anything` — use this by default
- Default from `kubectl create ingress` is `Exact` — change to `Prefix` if path matching fails

**Also expose containerPort on the deployment** if question says "expose container port":
```bash
kubectl edit deployment <name> -n <ns>
# add under containers[0]:
# ports:
# - containerPort: 80
#   protocol: TCP
```

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

**Sidecar container — command array structure:**
```yaml
containers:
- name: sidecar
  image: busybox:stable
  command:
  - /bin/sh
  - -c
  - "touch /var/log/app.log && tail -f /var/log/app.log"  # one string — never split shell cmd across elements
  volumeMounts:
  - name: logs
    mountPath: /var/log
```

**Shared volume between main + sidecar:**
```yaml
volumes:
- name: logs
  emptyDir: {}
```

**Gotcha:** `tail -f <file>` crashes if file doesn't exist yet. Always `touch` the file first.

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

**Always include `-n <namespace>` in the imperative command** so it's baked into the YAML:
```bash
kubectl autoscale deployment apache-server -n autoscale --min=1 --max=4 --cpu-percent=50 --dry-run=client -o yaml > hpa.yaml
```

**HPA behavior block** — add after metrics section:
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 30   # wait 30s before scaling down
  scaleUp:
    stabilizationWindowSeconds: 0    # scale up immediately
```

`behavior` → `scaleDown`/`scaleUp` → `stabilizationWindowSeconds` — three levels deep.

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

**Identify user-defined vs system PriorityClass:**
- User-defined → normal values (100, 1000, etc.)
- System built-in → values in the billions (`2000000000`+): `system-cluster-critical`, `system-node-critical`

**`priorityClassName` goes in `spec.template.spec`** of a Deployment:
```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: app
```

**Calculate value:** `kubectl get priorityclass` → find highest user-defined value → subtract 1.

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

**Pending pod due to Insufficient cpu/memory — quick fix for exam:**
```bash
# 1. Check why pending
kubectl describe pod <pod> -n <ns>   # scroll to Events: at bottom

# 2. Fix — reduce resource requests to safe values
kubectl edit deployment <name> -n <ns>
# set cpu: 250m  memory: 256Mi  → apply → verify pods Running
```

**Triage order for pending pods:**
1. `kubectl describe pod` → Events → read the reason
2. `Insufficient cpu/memory` → reduce requests
3. `taint` related → check tolerations / nodeSelector
4. Don't check taints/nodeSelector first unless events say so

**Resource math (if exam asks to calculate fairly):**
```bash
# Convert: 1 CPU = 1000m | Ki → Mi: divide by 1024 | Mi → Gi: divide by 1024
# bc doesn't handle decimals — use * 90 / 100 instead of * 0.9

kubectl describe node <node> | grep -A6 Allocatable
kubectl describe node <node> | grep -A10 'Allocated resources'

# CPU remaining with 10% buffer divided by pending pods:
echo "(2000 - 1150) * 90 / 100 / 2" | bc
# Memory (convert Ki→Mi first, then same pattern):
echo "(1857 - 900) * 90 / 100 / 2" | bc
```

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

# Find values to override — grep for exact key names
helm show values bitnami/nginx | grep -Ei 'replica|type'

# Check installed release
helm list -n <ns>
helm get metadata web-release -n <ns>   # shows chart name + version

# Upgrade — re-specify all values or use --reuse-values
helm upgrade web-release bitnami/nginx -n <ns> --set replicaCount=3 --set service.type=NodePort

# Better: reuse previous values, only override what changed
helm upgrade web-release bitnami/nginx -n <ns> --reuse-values --set replicaCount=3

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

**Bind PVC directly to existing PV (Retain policy recovery):**
```yaml
spec:
  storageClassName: ""      # empty string = bypass StorageClass, bind directly to PV
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi          # must match PV capacity exactly
```
Check PV details first: `kubectl get pv <name> -o yaml` — match accessModes, capacity, and leave storageClassName empty.

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

**Set StorageClass as default:**
```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # string "true" not boolean
```

**Remove default from existing StorageClass** (if replacing):
```bash
kubectl annotate sc <old-sc> storageclass.kubernetes.io/is-default-class-
# trailing - removes the annotation
```

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
| HPA behavior nesting | `behavior` → `scaleDown` → `stabilizationWindowSeconds` — three levels deep |
| HPA imperative missing `-n` | Always add `-n <ns>` in `kubectl autoscale` so namespace is baked into YAML |
| Helm upgrade loses values | Use `--reuse-values` to keep previous values, only override what changed |
| StorageClass default | Set via annotation `storageclass.kubernetes.io/is-default-class: "true"` — not a field |
| Remove annotation | `kubectl annotate sc <name> <key>-` — trailing `-` removes the annotation |
| PVC bind to existing PV | Set `storageClassName: ""` (empty string) to bypass StorageClass and bind directly |
| CRD metadata.name | Must be `<plural>.<group>` — e.g. `backups.stable.example.com` |
| CRD schema required | Cannot omit `openAPIV3Schema`. Minimum: `openAPIV3Schema: type: object` |
| ConfigMap immutable | `immutable: true` at top level (not under data). Delete+recreate to change values after |
| Ingress pathType default | `kubectl create ingress` defaults to `Exact`. Change to `Prefix` if path matching fails |
| Ingress missing --class | Always add `--class=nginx`. Without it ingress has no controller and won't work |
| Expose containerPort | Question says "expose container port" = add `ports.containerPort` to deployment spec too |
| Gateway API NodePort | Find NodePort on the Gateway's own service in your namespace, not the controller namespace |
| Cluster upgrade drain order | Control plane: upgrade kubeadm → apply → drain → upgrade kubelet. Worker: drain first |
| Cluster upgrade worker kubectl | Don't install kubectl on worker nodes — not needed, only kubeadm + kubelet |
| Sidecar command array | `/bin/sh`, `-c`, `"full command"` — entire shell command is ONE string as third element |
| Sidecar file not found | `tail -f` crashes if file missing. Use `touch <file> && tail -f <file>` |
| User-defined PriorityClass | Normal values (100-10000). System classes have values in billions (2000000000+) |
| Pending pod triage | Check `Events:` in `describe pod` first — it tells you exactly why. Don't guess taints/selectors |

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

## Gateway API

```yaml
# 1. Create Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: <ns>
spec:
  gatewayClassName: nginx       # check with: kubectl get gatewayclass
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: gateway.web.k8s.local

---
# 2. Create HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: <ns>
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - backendRefs:
    - name: web-service
      port: 80
```

**Key difference from Ingress:** Gateway API provisions its own **dedicated dataplane pod + NodePort service** in your namespace when you create a Gateway. Find the NodePort on that service — not on the controller namespace:
```bash
kubectl get svc -n <ns>   # look for <gateway-name>-nginx NodePort service
```

**Verify:** `curl -H 'Host: gateway.web.k8s.local' http://<node-ip>:<nodeport>`

---

## ConfigMap

```bash
# Create
kubectl create configmap app-config -n <ns> \
  --from-literal=LOG_LEVEL=info \
  --from-literal=APP_ENV=production

# Edit a value
kubectl edit cm app-config -n <ns>

# Make immutable (cannot be edited after — pods must be restarted to pick up changes)
kubectl edit cm app-config -n <ns>
# add at top level: immutable: true

# Verify field
kubectl explain cm.immutable

# Restart deployment to pick up new ConfigMap values
kubectl rollout restart deployment <name> -n <ns>
```

**Immutable ConfigMap YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true        # top level, not under data
data:
  LOG_LEVEL: debug
```

**Gotcha:** Once `immutable: true` is set, any edit attempt is rejected: `field is immutable when immutable is set`. To change values you must delete and recreate the ConfigMap, then restart pods.

---

## CRD — CustomResourceDefinition

**Minimum valid CRD:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com   # MUST be <plural>.<group>
spec:
  group: stable.example.com
  scope: Namespaced                  # or Cluster
  names:
    plural: backups
    singular: backup
    kind: Backup
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object                 # minimum required — cannot omit openAPIV3Schema
```

**Create a CR instance after CRD is created:**
```yaml
apiVersion: stable.example.com/v1
kind: Backup
metadata:
  name: my-backup
  namespace: <ns>
spec: {}
```

**Inspect CRD fields:**
```bash
kubectl explain backup
kubectl explain backup.spec
```

**Exam tip:** Copy from docs and strip back to minimum. The only required schema is `openAPIV3Schema: type: object`.

---

## CNI

**Where CNI config lives on each node:**
```bash
ls /etc/cni/net.d/
# e.g. 10-calico.conflist, 10-flannel.conflist
```

**Check pod network CIDR:**
```bash
kubectl describe node | grep -i cidr
```

**Check pod IP assignment:**
```bash
kubectl get pods -o wide              # fastest — shows IP column
kubectl describe pod <pod> -n <ns>    # more detail — Events show CNI errors
```

**If pod has no IP:** Events will say `Failed to create pod sandbox: CNI plugin not initialized` → CNI is broken or not installed.

**After CNI issues — restart order:**
```bash
kubectl rollout restart daemonset kube-proxy -n kube-system    # fix ClusterIP routing first
kubectl rollout restart daemonset calico-node -n kube-system   # fix pod network second
```

---

## Cluster Upgrade

**Full upgrade procedure — control plane first, then workers:**

```bash
# === CONTROL PLANE ===

# 1. Add new version repo
sudo vi /etc/apt/sources.list.d/kubernetes.list
# change v1.33 → v1.34

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm='1.34.0-1.1'
sudo apt-mark hold kubeadm

# 3. Check upgrade plan
sudo kubeadm upgrade plan

# 4. Apply upgrade (pre-pull images first to avoid timeout)
sudo kubeadm config images pull --kubernetes-version v1.34.0
sudo kubeadm upgrade apply v1.34.0

# 5. Drain control plane
kubectl drain k8s-master --ignore-daemonsets

# 6. Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.34.0-1.1' kubectl='1.34.0-1.1'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# 7. Uncordon
kubectl uncordon k8s-master

# === WORKER NODE ===
# (drain from master first, then ssh to worker)

kubectl drain k8s-worker --ignore-daemonsets   # run from master

# ssh to worker:
sudo vi /etc/apt/sources.list.d/kubernetes.list   # change repo version
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm='1.34.0-1.1'
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet='1.34.0-1.1'   # kubectl not needed on worker
sudo apt-mark hold kubelet
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# back on master:
kubectl uncordon k8s-worker

# Verify
kubectl get nodes   # both nodes should show new version
```

**Key order:** `kubeadm upgrade apply` THEN drain THEN upgrade kubelet (not drain first on control plane).
**Worker:** drain from master FIRST, then upgrade kubeadm + kubelet on worker.
**Verify kubelet version:** `kubelet --version` on each node.

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
