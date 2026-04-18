# CKA Cheatsheet: Services & Networking, Workloads & Scheduling, Storage, Cluster Admin

Covers domains: Services & Networking (20%), Workloads & Scheduling (15%), Storage (10%), Cluster Architecture (25%)

---

## Quick Setup

```bash
# vim settings — add to ~/.vimrc
set ts=2 sw=2 et

# Session helpers — paste after each ssh if needed
kns(){ kubectl config set-context --current --namespace="$1"; }
kctx(){ kubectl config current-context; }
kg(){ kubectl get "$@"; }
kdes(){ kubectl describe "$@"; }
kl(){ kubectl logs "$@"; }
kdr(){ kubectl "$@" --dry-run=client -o yaml; }

# Check current namespace
kubectl config view --minify | grep namespace

# Package version troubleshooting on Ubuntu/Debian nodes
sudo apt update
apt-cache policy kubeadm
apt-cache policy kubelet
apt-cache policy kubectl
apt list -a kubeadm
apt list -a kubelet
apt list -a kubectl
apt list --upgradable
```

### Context Switching

```bash
# Show contexts
kubectl config get-contexts
kubectl config current-context

# Switch to another context
kubectl config use-context cluster3

# Change namespace on current context
kubectl config set-context --current --namespace=kube-system

# Run one command against another context without switching permanently
kubectl get pods --context=cluster4 -A
```

**Memory trick:**
- `use-context` = move to it
- `set-context` = edit it

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

### Service Without Selector + Manual EndpointSlice

Use this when the backend is **not a Pod**, for example an external host/IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-webserver
  namespace: kube-public
spec:
  ports:
  - name: http
    port: 80
    targetPort: 9999

---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-webserver-1
  namespace: kube-public
  labels:
    kubernetes.io/service-name: external-webserver
    endpointslice.kubernetes.io/managed-by: staff
addressType: IPv4
ports:
- name: http
  protocol: TCP
  port: 9999
endpoints:
- addresses:
  - 10.244.222.57
  conditions:
    ready: true
```

**Gotchas:**
- Service must have **no selector**
- `kubernetes.io/service-name` label must match the Service name
- Service port name and EndpointSlice port name should match
- EndpointSlice name itself can be anything unique
- Pod resolves the **Service name** to ClusterIP; kube-proxy forwards to the EndpointSlice IP:port
- `ExternalName` is different: use it for DNS aliasing, not forwarding to an external IP:port

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

**Useful DNS patterns to remember:**
- Service FQDN:
  - `service.namespace.svc.cluster.local`
- Pod with `hostname` + `subdomain`:
  - `<hostname>.<subdomain>.<namespace>.svc.cluster.local`
  - example: `section100.section.default.svc.cluster.local`
- Pod dashed-IP DNS:
  - `podip-with-dashes.namespace.pod.cluster.local`
  - example: `172-17-2-5.default.pod.cluster.local`
- StatefulSet Pod via headless Service:
  - `podname.headless-service.namespace.svc.cluster.local`
  - example: `web-0.nginx.default.svc.cluster.local`

**Exam note:**
- Use Service DNS for normal app access.
- Pod-specific `hostname.subdomain.ns.svc.cluster.local` requires a Service whose name matches the `subdomain`. For stable per-Pod DNS, use a headless Service.
- Pod DNS by dashed IP can work, but do not rely on the shorter `podip.namespace` form unless you verify it in the cluster.
- For cross-namespace access, use at least `service.namespace`, or safest: `service.namespace.svc.cluster.local`
- BusyBox `nslookup` can be misleading; trust actual connectivity tests like `wget`/`curl` if FQDN access works

### Name Resolution Helpers

```bash
# Query system NSS database for host resolution
getent hosts student-node

# Show local host IPs
hostname -I

# Show preferred source IP used for outbound traffic
ip route get 1.1.1.1
```

**Notes:**
- `getent` queries OS lookup databases through NSS (`/etc/nsswitch.conf`)
- `hosts` database can resolve via `/etc/hosts`, DNS, or other configured sources

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

**Shared `emptyDir` mental model:**
- same Pod only
- same underlying volume
- containers can mount it at **different paths** and still see the same files
- example: `/log/app.log` in one container can be `/usr/share/nginx/html/app.log` in another

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

**High-value shapes to remember:**
- Pod name via downward API:
  ```yaml
  fieldRef:
    fieldPath: metadata.name
  ```
- Node name via downward API:
  ```yaml
  fieldRef:
    fieldPath: spec.nodeName
  ```
- Shared per-Pod scratch volume:
  ```yaml
  volumes:
  - name: shared
    emptyDir: {}
  ```
- If the question says a shared volume is mounted into each container, verify all containers have `volumeMounts`.

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

**Syntax memory:**
- `nodeSelector` = plain map only
  ```yaml
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  ```
- `nodeAffinity` = `nodeSelectorTerms` + `matchExpressions`
- `matchLabels` = simple `key: value`
- `matchExpressions` = `key`, `operator`, optional/required `values`

**Operator rule:**
- `In`, `NotIn`, `Gt`, `Lt` -> use `values`
- `Exists`, `DoesNotExist` -> do not use `values`

### DaemonSet Toleration Shortcut

```yaml
tolerations:
- operator: Exists
```

Useful for DaemonSets that should run on broadly tainted nodes, including control-plane nodes. It is a broad toleration shortcut.

**Control-plane-only pod pattern (kubeadm-style):**
- Restrict to control-plane nodes:
  ```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
  ```
- Also tolerate the taint:
  ```yaml
  tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  ```
- `nodeName` pins to one exact node. It does not express a scheduling rule.

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

**Version gotcha:**
- `kubectl autoscale` may try `autoscaling/v2` first, but can still fall back to `autoscaling/v1`
- `autoscaling/v1` uses:
  ```yaml
  targetCPUUtilizationPercentage: 65
  ```
- `autoscaling/v2` uses:
  ```yaml
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  ```
- `behavior` works only in `autoscaling/v2`
- Exam-safe rule: for advanced HPA questions, write `autoscaling/v2` manually

**Advanced HPA (`autoscaling/v2`) examples:**

```yaml
# CPU + behavior
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: cka0841
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      selectPolicy: Min
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60
      - type: Percent
        value: 20
        periodSeconds: 60
```

```yaml
# Pods custom metric
metrics:
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

```yaml
# Object custom metric
metrics:
- type: Object
  object:
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: web-ingress
    metric:
      name: requests_per_second
    target:
      type: Value
      value: "2000"
```

**HPA behavior notes:**
- `selectPolicy: Min` = use the smallest allowed change
- `selectPolicy: Max` = use the largest allowed change
- Default policy selection behaves like `Max`
- Default stabilization windows commonly seen:
  - `scaleUp.stabilizationWindowSeconds: 0`
  - `scaleDown.stabilizationWindowSeconds: 300`
- `stabilizationWindowSeconds` smooths scaling decisions and helps avoid flapping

**Custom metrics mental model:**
- `Resource` = CPU/memory
- `Pods` = metric per pod, usually `AverageValue`
- `Object` = metric on one K8s object, usually `Value`
- `External` = metric outside Kubernetes
- HPA does **not** read Prometheus directly:
  `Prometheus -> prometheus-adapter -> custom.metrics/external.metrics API -> HPA`

### VPA

**Install note:** CRD alone is not enough. VPA needs:
- recommender
- updater
- admission-controller

```bash
kubectl get pods -n kube-system | grep vpa
kubectl get vpa -A
```

**`Initial` mode gotchas:**
- `Initial` only applies recommendations to **new Pods**
- if `CPU`/`MEM` columns are empty, VPA has no recommendation yet
- restarting workload before recommendation exists does nothing
- if the question only asks to create the VPA, you usually do not need to restart anything
- if recommendation appears and task wants it applied, recreate/restart the Pods

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cache-vpa
  namespace: caching
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: cache-statefulset
  updatePolicy:
    updateMode: Initial
```

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

**Fast memory:**
- list repos:
  ```bash
  helm repo list
  ```
- repo already exists and install works -> `helm repo update` is optional
- `helm search repo` searches added repos
- `helm search hub` is slower discovery; use only if you do not know the repo yet

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
- `storageClassName` must match exactly (string equality)
- `accessModes` must be compatible (PVC modes ⊆ PV modes)
- PVC `storage` request must be **≤** PV capacity (not required to be equal)

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
Check PV details first: `kubectl get pv <name> -o yaml` — match accessModes and storageClassName. PVC request must be **≤ PV capacity** (not necessarily equal).

**Default vs manual habit:**
- Default StorageClass + dynamic provisioning:
  - omit `storageClassName`
  - do not create a PV manually
- Manual PV/PVC binding:
  - set `storageClassName: ""`
  - optionally pin with `volumeName: <pv-name>` for exam-safe explicit binding
- If a task creates a StorageClass with `provisioner:`, think dynamic provisioning first.

**Diagnose PVC stuck Pending:**
```bash
kubectl describe pvc <name> -n <ns>
# Check: Events — usually "no persistent volumes available" or "storageClass mismatch"
kubectl get pv   # check PV status and storageClassName
```

### `hostPath` vs `local`

```yaml
# hostPath PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/hostpath-demo

---
# local PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  local:
    path: /data/local-demo
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-name
```

**Mental model:**
- `hostPath` = simple host filesystem path mount on whichever node the Pod runs
- `local` = node-bound local disk PV, must declare ownership with `nodeAffinity`
- If the question says `hostPath`, use `hostPath` — do not switch to `local`
- Both point at real node filesystem paths; deleting the Pod/PV object does not automatically remove host files
- `hostPath.type` is mount-time behaviour:
  - omitted = use path as-is
  - `Directory` = path must already exist
  - `DirectoryOrCreate` = create directory if missing when mounted
- `persistentVolumeReclaimPolicy` can be set on both, but it does not usually delete raw node directories for `hostPath`/`local`

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

**Important nuance:** if a PVC sets `volumeName` to a specific PV, binding can still succeed without creating a consumer pod, as long as the rest of the PV/PVC fields match.

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

### PV/PVC Matching Extras

**PVC can constrain matching with all of these at once:**
- `storageClassName`
- `accessModes`
- requested capacity `<=` PV capacity
- `volumeMode`
- `selector.matchLabels` against PV labels
- `volumeName` for a specific PV

**Selector belongs on PVC, not PV:**
```yaml
# PV
metadata:
  labels:
    storage-tier: gold

# PVC
spec:
  selector:
    matchLabels:
      storage-tier: gold
```

**Force a specific PV:**
```yaml
spec:
  storageClassName: ""
  volumeName: peach-pv-cka05-str
```

**Notes:**
- `selector` on a PV is invalid
- `volumeName` on PVC points to an exact PV
- if PVC uses both `volumeName` and `selector`, the chosen PV must satisfy both
- omitting `volumeName` lets Kubernetes choose any suitable PV
- for PVs with empty storage class, safest PVC form is `storageClassName: ""`

**Released PV recovery (`Retain` policy):**
```bash
kubectl patch pv peach-pv-cka05-str --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
```

`Released` usually means the old `claimRef` is still present. Remove it to make the PV reusable.

**Node-specific PVs:**
```yaml
spec:
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - cluster1-controlplane
```

Use `kubectl describe pv <name>` or `kubectl get pv <name> -o yaml` to verify the node affinity.

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

**Memory trick:**
- `RoleBinding --serviceaccount` format: `<namespace>:<sa-name>`
- `kubectl auth can-i --as` for ServiceAccount: `system:serviceaccount:<namespace>:<sa-name>`
- For namespaced RBAC checks, include `-n <ns>`
- If resources belong to different API groups in raw YAML, split them into separate `rules:` entries.

**Common API group mapping:**
- core resources like `pods`, `configmaps`, `secrets` -> `apiGroups: [""]`
- `deployments`, `daemonsets`, `statefulsets` -> `apiGroups: ["apps"]`

### Preview Changes

```bash
kubectl diff -f file.yaml
```

Shows what `kubectl apply -f file.yaml` would change, without applying it. Useful before recreating or replacing broken objects.

### Optional `yq`

```bash
# Read field
yq '.metadata.name' file.yaml

# Edit in place
yq -i '.spec.replicas = 3' file.yaml

# Delete field
yq -i 'del(.status)' file.yaml

# Inspect kubectl YAML
kubectl get pod x -o yaml | yq '.status.phase'
```

Nice to have, not required for CKA.

### ResourceQuota

```bash
kubectl create quota my-quota -n <ns> \
  --hard=pods=10,cpu=2,memory=1Gi

kubectl describe resourcequota my-quota -n <ns>
```

### Probe Defaults

**High-value memory:**
- `timeoutSeconds: 1`
- `periodSeconds: 10`
- `failureThreshold: 3`
- `successThreshold: 1`
- `initialDelaySeconds: 0`

**Practice rule:**
- If using `exec` with `wget`, `curl`, or shell logic, default `timeoutSeconds: 1` is often too short.
- If the prompt gives a shell command, prefer `exec` over forcing `httpGet`.

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
| PVC capacity "must match exactly" | Wrong — PVC request must be **≤** PV capacity. 1Gi PVC can bind to a 5Gi PV |
| Drain without `--delete-emptydir-data` | Drain hangs waiting for pods with emptyDir volumes. Always include the flag |
| init container failures | Pod stays `Init:0/N` — check `kubectl logs <pod> -c <init-container-name>` |
| `kubectl rollout restart` vs delete pods | Use restart for rolling pod replacement. Deleting pods manually causes downtime |
| `kubectl rollout pause` forgot to resume | Changes accumulate but are never applied. Always `kubectl rollout resume` after pausing |
| DaemonSet on control plane | Not scheduled there by default — add toleration for `node-role.kubernetes.io/control-plane:NoSchedule` |
| `commonLabels` in Kustomize | Modifies selector labels too — breaks existing Deployments if added after creation |
| Kustomize hash suffix | Generated ConfigMaps/Secrets get hash suffix by default — use `disableNameSuffixHash: true` to suppress |
| `kubectl apply -k` vs `kubectl kustomize` | `apply -k` deploys; `kustomize` only renders. They are not interchangeable |
| `--as=<sa-name>` for ServiceAccount | Use `--as=system:serviceaccount:<ns>:<name>`. Bare name creates a User binding |
| ClusterRoleBinding vs RoleBinding + ClusterRole | ClusterRoleBinding = cluster-wide access. RoleBinding + ClusterRole = namespace-scoped access |
| `kubectl auth can-i` missing `-n` | Namespaced Role/RoleBinding checks can look wrong without `-n <ns>` |
| etcd restore partial manifest update | Must update `--data-dir`, `mountPath`, AND `hostPath.path` — all three must match |
| `kubeadm certs renew` without restart | New certs on disk but components still use old in-memory certs. Always restart kubelet after |
| `readOnlyRootFilesystem: true` breaks /tmp writes | Mount an `emptyDir` at `/tmp` for apps that need a writable temp dir |
| `fsGroup` at container level | `fsGroup` is pod-level only — cannot be set per container |
| CSI driver missing = PVC Pending | Dynamic PVCs won't bind if the CSI driver pod is not running |
| CronJob 100+ missed runs | CronJob stops scheduling if 100+ runs were missed. Check `startingDeadlineSeconds` |

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

### HTTPRoute Patterns

**Weighted traffic split:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-portal-httproute
  namespace: cka3658
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-portal-service-v1
      port: 80
      weight: 80
    - name: web-portal-service-v2
      port: 80
      weight: 20
```

**Header-based canary route:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-app-route
  namespace: ck2145
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - headers:
      - name: X-Environment
        value: canary
    backendRefs:
    - name: web-service-canary
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 8080
```

**HTTPRoute notes:**
- `hostnames` is optional; if omitted, route is not restricted by Host header
- `hostnames` matches the HTTP `Host` header
- `headers` matches other headers, for example `X-Environment: canary`
- if one `match` contains both `path` and `headers`, both must match
- multiple header entries in one match are ANDed
- put the most specific rule first for clarity
- test header routes with:
  ```bash
  curl -H 'X-Environment: canary' http://localhost:30080
  ```

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

**ConfigMap env / volume shapes:**
- Inject all keys as env vars:
  ```yaml
  envFrom:
  - configMapRef:
      name: app-config
  ```
- Inject one key to one env var:
  ```yaml
  env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: mode
  ```
- Mount as volume:
  ```yaml
  volumes:
  - name: cfg
    configMap:
      name: app-config
  ```
- Mounted ConfigMap path is a directory; keys become files inside it.

**Secret env / volume shapes:**
- Inject all keys as env vars:
  ```yaml
  envFrom:
  - secretRef:
      name: app-secret
  ```
- Inject one key to one env var:
  ```yaml
  env:
  - name: APP_PASS
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: password
  ```
- Mount as volume:
  ```yaml
  volumes:
  - name: sec
    secret:
      secretName: app-secret
  ```
- Secret mount path is a directory; keys become files inside it.
- If the question says the Secret mount is read-only, add:
  ```yaml
  readOnly: true
  ```

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
kubectl drain k8s-master --ignore-daemonsets --delete-emptydir-data

# 6. Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.34.0-1.1' kubectl='1.34.0-1.1'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# 7. Uncordon
kubectl uncordon k8s-master

# === WORKER NODE ===
# (drain from master first, then ssh to worker)

kubectl drain k8s-worker --ignore-daemonsets --delete-emptydir-data   # run from master

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

## Kustomize

```bash
# Apply with kustomize (replaces kubectl apply -f)
kubectl apply -k <dir>

# Preview what will be applied (does NOT apply)
kubectl kustomize <dir>

# Diff against live cluster
kubectl diff -k <dir>

# Dry run
kubectl apply -k <dir> --dry-run=client -o yaml
```

### kustomization.yaml — Exam-Ready Template

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production          # override namespace on all resources
namePrefix: prod-              # prepend to all resource names
nameSuffix: -v2

commonLabels:
  env: prod                    # added to ALL resources (including selectors!)

resources:
- deployment.yaml
- service.yaml
- ../base                      # reference base layer for overlay pattern

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
  - APP_ENV=production

secretGenerator:
- name: app-secret
  literals:
  - DB_PASS=s3cr3t

generatorOptions:
  disableNameSuffixHash: true  # suppress hash suffix on generated ConfigMaps/Secrets
```

**Gotcha:** `commonLabels` modifies `spec.selector` labels too. Don't add it to existing Deployments — it will break the selector match.

**Gotcha:** Generated ConfigMaps/Secrets get a content-hash suffix (e.g. `app-config-8dk7t`) by default. Use `generatorOptions.disableNameSuffixHash: true` to suppress.

**Gotcha:** `kubectl apply -k .` deploys. `kubectl kustomize .` only renders. They are not the same.

---

## Jobs and CronJobs

### Job — Key Fields

```bash
# Imperative
kubectl create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(2000)'
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 6            # total successful pod completions needed
  parallelism: 2            # max pods running simultaneously
  backoffLimit: 3           # retries before marking Job Failed (default 6)
  activeDeadlineSeconds: 60 # hard timeout for the entire job
  template:
    spec:
      restartPolicy: OnFailure   # Never or OnFailure — NEVER use Always
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo done"]
```

```bash
kubectl get jobs
kubectl describe job <name>
kubectl logs <job-pod-name>       # get pod name from: kubectl get pods --selector=job-name=<name>
```

**Gotcha:** `restartPolicy: Always` is forbidden in Jobs — pod will be rejected.

### CronJob — Key Fields

```bash
# Imperative
kubectl create cronjob hello --image=busybox --schedule="*/5 * * * *" -- date
```

```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 2 * * *"           # cron format: min hour dom mon dow
  concurrencyPolicy: Forbid        # Allow | Forbid | Replace
  startingDeadlineSeconds: 60      # skip if can't start within 60s of scheduled time
  suspend: false                   # true = pause without deleting
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: job
            image: busybox
            command: ["sh", "-c", "date"]
```

```bash
# Trigger immediately (without waiting for schedule)
kubectl create job --from=cronjob/<name> manual-run-$(date +%s)

# Suspend/resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'
```

| `concurrencyPolicy` | Behaviour |
|---|---|
| `Allow` (default) | New job starts even if previous still running |
| `Forbid` | Skip new run if previous job still running |
| `Replace` | Cancel previous job, start new one |

---

## DaemonSet

Runs exactly **one pod per eligible node**. Used for logging agents, monitoring, CNI components.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-agent
  updateStrategy:
    type: RollingUpdate        # or OnDelete
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule     # run on control plane nodes too
      containers:
      - name: agent
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

```bash
kubectl get ds -A                           # list all DaemonSets
kubectl rollout status daemonset/<name>
kubectl rollout undo daemonset/<name>
kubectl set image daemonset/<name> <container>=<new-image>
```

**Gotcha:** DaemonSets have no `replicas` field — pod count equals eligible node count.

**Gotcha:** DaemonSets don't run on control plane nodes by default — add the toleration above if needed.

**Update strategies:**
- `RollingUpdate` (default) — old pods replaced progressively
- `OnDelete` — new pods only created when old pods are manually deleted

---

## Init Containers

Run sequentially before app containers start. All must succeed for app containers to launch.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-svc 5432; do echo waiting; sleep 2; done']
  - name: init-config
    image: busybox
    command: ['sh', '-c', 'echo "config" > /shared/config.txt']
    volumeMounts:
    - name: shared
      mountPath: /shared
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: shared
      mountPath: /config
  volumes:
  - name: shared
    emptyDir: {}
```

```bash
# Logs from init container
kubectl logs <pod> -c wait-for-db
kubectl logs <pod> -c wait-for-db --previous

# Init container status
kubectl describe pod <pod>   # "Init Containers:" section
```

**Gotcha:** Pod stays in `Init:0/2` status if any init container fails or hasn't completed. Check init container logs first.

**Gotcha:** Init containers run sequentially — `initContainers[0]` must complete before `initContainers[1]` starts.

---

## Security Context

### Pod-level vs Container-level

```yaml
spec:
  securityContext:                 # pod-level — applies to ALL containers
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000                  # volume ownership GID
    runAsNonRoot: true
  containers:
  - name: app
    image: nginx
    securityContext:               # container-level — overrides pod-level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

| Field | Level | Effect |
|---|---|---|
| `runAsUser` | Pod or Container | UID for the process |
| `runAsGroup` | Pod or Container | GID for the process |
| `fsGroup` | Pod only | GID for volume file ownership |
| `runAsNonRoot` | Pod or Container | Reject pod if UID = 0 |
| `allowPrivilegeEscalation` | Container only | Block setuid/capability escalation |
| `readOnlyRootFilesystem` | Container only | Mount root filesystem read-only |
| `capabilities.drop/add` | Container only | Linux capability control |

**Gotcha:** `fsGroup` is pod-level only — cannot be set per container.

**Gotcha:** `readOnlyRootFilesystem: true` will break apps that write to `/tmp` — mount an `emptyDir` for writable paths.

---

## RBAC — Full Creation Pattern

```bash
# Create Role (namespace-scoped)
kubectl create role pod-reader -n <ns> \
  --verb=get,list,watch \
  --resource=pods

# Bind Role to a User
kubectl create rolebinding alice-pod-reader -n <ns> \
  --role=pod-reader \
  --user=alice

# Bind Role to a ServiceAccount
kubectl create rolebinding sa-pod-reader -n <ns> \
  --role=pod-reader \
  --serviceaccount=<ns>:<sa-name>

# Create ClusterRole (cluster-wide)
kubectl create clusterrole node-viewer \
  --verb=get,list \
  --resource=nodes

# Bind ClusterRole cluster-wide
kubectl create clusterrolebinding node-viewer-binding \
  --clusterrole=node-viewer \
  --user=alice

# Bind ClusterRole but only for one namespace (RoleBinding)
kubectl create rolebinding alice-node-viewer -n <ns> \
  --clusterrole=node-viewer \
  --user=alice
```

```bash
# Create ServiceAccount
kubectl create serviceaccount <sa-name> -n <ns>

# Verify permissions
kubectl auth can-i list pods -n <ns> --as=alice
kubectl auth can-i list pods -n <ns> --as=system:serviceaccount:<ns>:<sa-name>
kubectl auth can-i --list --as=alice -n <ns>    # show everything alice can do
```

**Gotcha:** `--serviceaccount=<ns>:<sa-name>` — namespace:name format, separated by colon. `--user=<sa-name>` creates a User binding, which is different.

**Gotcha:** A `ClusterRoleBinding` gives cluster-wide access. A `RoleBinding` referencing a `ClusterRole` gives access only in that namespace.

### In-Pod API Curl with ServiceAccount

```bash
# Files mounted automatically when Pod uses a ServiceAccount
/var/run/secrets/kubernetes.io/serviceaccount/token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace

# Create ConfigMap through Kubernetes API from inside a Pod, write response on host
kubectl exec -n rbac-lab api-check -- sh -c '
APISERVER=https://kubernetes.default.svc
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
TOKEN=$(cat ${SERVICEACCOUNT}/token)
CACERT=${SERVICEACCOUNT}/ca.crt
curl --cacert "${CACERT}" \
  --header "Authorization: Bearer ${TOKEN}" \
  --header "Content-Type: application/json" \
  -X POST "${APISERVER}/api/v1/namespaces/${NAMESPACE}/configmaps" \
  -d '"'"'{
    "apiVersion":"v1",
    "kind":"ConfigMap",
    "metadata":{"name":"api-created"},
    "data":{"source":"api"}
  }'"'"'
' > /tmp/rbac-create-configmap.json
```

**Notes:**
- `>` must stay outside `kubectl exec ... sh -c '...'` if the file should be written on the host
- For Secrets, `data` values must be base64-encoded strings
- Multi-line `curl` inside `sh -c` needs `\` line continuations

**Quote timeline (`'"'"'"'"'"'"'"'"'`):**
```text
What you type:
sh -c 'curl ... -d '"'"'hello'"'"''

Shell pieces:
'curl ... -d '   -> curl ... -d
"'"              -> '
'hello'          -> hello
"'"              -> '
''               -> nothing

Final reconstructed text:
curl ... -d 'hello'
```

---

## kubectl rollout — Full Reference

```bash
# Watch rollout progress
kubectl rollout status deployment/<name> -n <ns> -w

# Show revision history
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns> --revision=2

# Undo last rollout
kubectl rollout undo deployment/<name> -n <ns>

# Undo to specific revision
kubectl rollout undo deployment/<name> -n <ns> --to-revision=2

# Pause rollout (accumulate changes without applying)
kubectl rollout pause deployment/<name> -n <ns>

# Make multiple changes while paused
kubectl set image deployment/<name> <c>=<new-image> -n <ns>
kubectl set resources deployment/<name> --requests=cpu=200m -n <ns>

# Resume — applies all accumulated changes at once
kubectl rollout resume deployment/<name> -n <ns>

# Restart all pods (rolling restart without image change)
kubectl rollout restart deployment/<name> -n <ns>
```

**Gotcha:** `rollout pause` + multiple changes + `rollout resume` = one combined rollout. Faster and safer than sequential updates.

**Gotcha:** `kubectl rollout restart` is the correct way to trigger a rolling restart. Do NOT delete pods manually — that causes downtime.

---

## kubectl debug — Ephemeral Containers

```bash
# Attach a debug container to a running pod (shares network namespace)
kubectl debug -it <pod> -n <ns> --image=busybox -- sh

# Debug a node (creates privileged pod on the node)
kubectl debug node/<node-name> -it --image=busybox

# Copy a broken pod with a different image for debugging
kubectl debug <pod> -n <ns> -it --copy-to=<new-pod-name> --image=busybox
```

**When to use:**
- Pod is in CrashLoopBackOff and you can't exec into it — use `--copy-to` with a shell image
- Need to test network from within the pod's network namespace — use the direct ephemeral approach
- Need node-level access — use `kubectl debug node/`

**Gotcha:** Ephemeral containers share the target pod's network namespace (localhost = target pod). They do NOT share filesystem unless you use `--share-processes`.

---

## Certificate Management

```bash
# Check all control plane cert expiry dates
sudo kubeadm certs check-expiration

# Renew all certs at once
sudo kubeadm certs renew all

# Renew a specific cert
sudo kubeadm certs renew apiserver

# CRITICAL — restart kubelet after renewal (certs loaded in memory must be refreshed)
sudo systemctl restart kubelet

# If kubectl also breaks (admin client cert expired), refresh kubeconfig:
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```

**Gotcha:** `kubeadm certs renew` writes new cert files to disk but does NOT reload running components. Always restart kubelet after renewal, or the old cert is still used in memory.

---

## CSI and CRI — Quick Reference

### CSI (Container Storage Interface)
```bash
kubectl get csidrivers                   # list installed CSI drivers
kubectl get sc                           # provisioner field shows which driver
kubectl get pods -A | grep csi           # check CSI driver pods

# If PVCs are stuck Pending and SC is dynamic:
# → CSI driver pod is probably missing or broken
kubectl describe pvc <name>             # Events will mention provisioner failure
```

### CRI (Container Runtime Interface)
```bash
# Check which runtime each node uses
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'

# On the node — use crictl (not docker)
sudo crictl ps -a                        # all containers including exited
sudo crictl logs <container-id>         # container logs
sudo crictl inspect <container-id>      # full config (image, env, mounts)
sudo crictl images                       # list images on node
```

**Gotcha:** `docker` CLI is gone in modern Kubernetes. Use `crictl` for all node-level container operations.

---

## etcd — Quick Reference

```bash
# Full backup command (cert paths from: grep cert /etc/kubernetes/manifests/etcd.yaml)
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd.db --write-out=table

# Restore snapshot to NEW directory
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd.db \
  --data-dir=/var/lib/etcd-restore

# After restore — edit /etc/kubernetes/manifests/etcd.yaml: THREE places
# 1. --data-dir=/var/lib/etcd-restore
# 2. volumeMounts.mountPath: /var/lib/etcd-restore
# 3. volumes.hostPath.path: /var/lib/etcd-restore
# Then wait 30s for kubelet to restart etcd static pod
```

**Gotcha:** Missing any one of the three manifest updates causes etcd to start with the old data directory and ignore the restore.

**Version check habit:**
- If etcd runs as a static Pod, get the version from the running etcd Pod:
  ```bash
  kubectl -n kube-system exec etcd-<node> -- etcd --version
  ```
- Do not install host packages just to answer a running-component version question.

## Kubelet Config / Cert Locations

```bash
# Kubelet client kubeconfig
/etc/kubernetes/kubelet.conf

# Kubelet config
/var/lib/kubelet/config.yaml

# Kubelet cert directory
/var/lib/kubelet/pki
```

**Common files:**
- client cert:
  - `/var/lib/kubelet/pki/kubelet-client-current.pem`
- server cert commonly:
  - `/var/lib/kubelet/pki/kubelet.crt`
- server key:
  - `/var/lib/kubelet/pki/kubelet.key`

**Inspect cert details:**
```bash
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -text
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -noout -text
```

**Mental model:**
- kubelet client cert = outgoing kubelet -> kube-apiserver
- kubelet server cert = incoming connections to kubelet
- `Extended Key Usage` confirms the role:
  - client auth vs server auth

---

## kubectl Output Formatting

```bash
# Wide output (more columns — shows node, IP)
kubectl get pods -o wide
kubectl get nodes -o wide

# YAML output (full resource spec)
kubectl get pod <name> -n <ns> -o yaml

# Extract a specific field with jsonpath
kubectl get pod <name> -o jsonpath='{.status.podIP}'
kubectl get pod <name> -o jsonpath='{.spec.containers[0].image}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# Multi-field jsonpath with range
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'

# Custom columns (tabular, readable)
kubectl get pods \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Sort output
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get events --sort-by=.metadata.creationTimestamp

# Filter with labels
kubectl get pods -l app=nginx,env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l 'env notin (dev)'

# All namespaces
kubectl get pods -A
```

---

## vim Tips

```text
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

Movement
h left, j down, k up, l right
w next word, b previous word
0 line start, $ line end
gg file top, G file bottom

Insert / edit
i insert before cursor
a insert after cursor
o new line below
O new line above
Esc back to normal mode

Delete
x delete character under cursor
dw delete forward one word
db delete backward one word
de delete to end of word
dd delete whole line
D delete from cursor to end of line

Undo / paste
u undo
Ctrl-r redo
yy yank line
p paste after cursor
P paste before cursor

Save / quit
:w save
:q quit
:wq save and quit
:q! quit without saving
```

**YAML paste recovery:**
```vim
:set list
:set expandtab
:retab
:set nolist
```
- `:set list` shows hidden chars (`^I` = tab)
- `:set expandtab` makes typed tabs become spaces
- `:retab` converts existing tabs to spaces
- `:set nolist` hides markers again

**Shift blocks quickly:**
```vim
V        " visual line select
j / k    " expand selection
>        " indent selected block one shiftwidth
<        " unindent selected block one shiftwidth

5>>      " indent current line + next 4 lines
5<<      " unindent current line + next 4 lines
```
- `V` then `j`/`k` is useful when you want to move a block elsewhere first
- `5>>` / `5<<` is faster when you already know roughly how many lines must shift
- `5>>` / `5<<` works without visual selection

### Shell Counting Pattern

```bash
kubectl get roles -A --no-headers | awk '{print $1}' | sort | uniq -c | sort -nr
```

**Meaning:**
- plain `sort` first groups identical lines together
- `uniq -c` counts adjacent duplicates
- `sort -nr` after that sorts by numeric count descending
