# CKA Notes: Workload and Scheduling

> **Domain weight: 15%.** Covers Deployments, DaemonSets, StatefulSets, Jobs, CronJobs, scheduling, autoscaling, probes, and security.

## Deployments

- Create: `kubectl create deploy web --image=nginx:1.25 --replicas=3`
- Scale: `kubectl scale deploy web --replicas=5`
- Verify: `kubectl get deploy,rs,pods`
- Update image: `kubectl set image deploy/web nginx=nginx:1.26`
- Watch rollout: `kubectl rollout status deploy/web`
- History: `kubectl rollout history deploy/web`
- Roll back: `kubectl rollout undo deploy/web`

Gotchas:
- `kubectl get pods` alone is not enough. Check Deployment, ReplicaSet, and Pods.
- Rollouts can be too fast on k3d. Use `kubectl get deploy,rs,pods -w`.

## Rolling Update Strategy

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

- `maxUnavailable=1`: at most 1 old pod can be unavailable during rollout.
- `maxSurge=1`: at most 1 extra new pod above desired replicas.

## ConfigMap

- Create:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=prod \
  --from-literal=LOG_LEVEL=info
```

- As env:

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV
```

- As volume:

```yaml
volumes:
- name: app
  configMap:
    name: app-config
```

Gotchas:
- ConfigMap as volume updates in running pods.
- ConfigMap as env var does not update until pod restart or recreate.
- ConfigMap volume files are symlinks through `..data`; that is normal.

## Secret

- Create:

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=redhat123
```

- `kubectl describe secret` shows byte counts, not cleartext values.
- Secret behavior follows the same pattern as ConfigMap:
- env vars are fixed at pod start
- mounted files can update

## Probes

- Readiness controls whether a pod receives traffic and whether rollout progresses.
- Liveness controls restarts.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
livenessProbe:
  httpGet:
    path: /
    port: 80
```

Gotchas:
- Bad readiness path: pod can be `Running` but `0/1 Ready`.
- Bad liveness path: pod restarts.
- Check with `kubectl describe pod <pod>`.

## HPA

- HPA needs CPU requests on target pods.
- Create:

```bash
kubectl autoscale deploy autoscale-web --cpu-percent=50 --min=1 --max=5
```

- Watch:

```bash
kubectl get hpa -w
```

Gotchas:
- Scheduler uses requests, not actual usage.
- Simple `nginx` may not generate enough CPU unless load is strong enough.
- Use a temporary load pod:

```bash
kubectl run -it --rm loadgen --image=busybox:1.36 --restart=Never -- sh
```

- Busybox usually has `wget`, not `curl`.
- `wget -O-` prints the response body to stdout.
- To discard the body:

```bash
while true; do wget -q -O /dev/null http://autoscale-service:80; done
```

## Jobs and CronJobs

- Job:

```bash
kubectl create job pi-job --image=perl:5.34 -- perl -e "print 'hello from job'"
```

- Verify:

```bash
kubectl get job pi-job
kubectl describe job pi-job
kubectl logs <job-pod>
```

- CronJob:

```bash
kubectl create cronjob hello-cron --image=busybox:1.36 --schedule="* * * * *" -- date
```

- Suspend and resume by editing `spec.suspend` or patching the object.

## Services, Endpoints, and EndpointSlices

- Expose deployment:

```bash
kubectl expose deploy api --name=api-svc --port=80
```

- Verify:

```bash
kubectl get svc
kubectl get endpoints api-svc
kubectl get endpointslices -l kubernetes.io/service-name=api-svc
```

Gotchas:
- `kubectl expose api` fails because `api` is not a resource type. Use `deploy api`.
- If Service selector is wrong, Service still exists but endpoints become `<none>`.
- `Endpoints` is deprecated, but still useful to know for troubleshooting.
- `EndpointSlice` is the modern replacement.

## Scheduling Basics

- Label node:

```bash
kubectl label node k3d-lab-agent-0 disk=ssd
```

- Node selector:

```yaml
nodeSelector:
  disk: ssd
```

- Taint:

```bash
kubectl taint nodes k3d-lab-agent-1 dedicated=teamA:NoSchedule
```

- Toleration:

```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: teamA
  effect: NoSchedule
```

Rules:
- Node selector and node affinity decide where a pod is allowed to run.
- Toleration does not force placement. It only allows scheduling onto a tainted node.

## Node Affinity

Hard rule:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disk
          operator: In
          values: [ssd]
```

Soft rule:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 80
  preference:
    matchExpressions:
    - key: zone
      operator: In
      values: [zone-a]
```

Gotchas:
- `required` is hard. If no node matches, pod stays `Pending`.
- `preferred` is tie-breaking among nodes that already satisfy hard rules.
- `nodeSelectorTerms` means OR between terms.
- `matchExpressions` inside one term means AND.

## Taints and Effects

- `NoSchedule`: blocks new non-tolerating pods.
- `NoExecute`: blocks new pods and evicts existing non-tolerating pods.

View taints:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

## Requests, Limits, and QoS

- Scheduler uses requests.
- CPU limits throttle.
- Memory limits can cause OOM kill.
- Use node **Allocatable**, not Capacity, for scheduling math.
- Actual runtime usage is not what the scheduler uses for placement.

Effective Pod request:
- regular containers: **sum** of container requests
- init containers: **max(highest init-container request, sum of regular container requests)** for each resource

QoS classes:
- `BestEffort`: no requests or limits
- `Burstable`: some requests or limits set, but not all equal
- `Guaranteed`: every container has CPU and memory requests and limits, and requests equal limits

Important subtlety:
- If CPU/memory `limits` are set and `requests` are omitted, the created Pod may end up with:
  - `requests == limits`
- Always inspect the created Pod, not only the YAML you wrote.

Check with:

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml
kubectl describe node <node>
```

## LimitRange and ResourceQuota

- `LimitRange` sets namespace defaults, mins, and maxes for resources.
- `ResourceQuota` limits total namespace usage.
- `ResourceQuota` is cumulative, not per pod.
- Admission failures show up as `FailedCreate` on the ReplicaSet/Deployment.
- Scheduler failures show up as `Pending` Pods with messages like:
  - `Insufficient cpu`
  - `Insufficient memory`
- Quota shorthand:
  - `requests.cpu` / `requests.memory` = requests quota
  - `limits.cpu` / `limits.memory` = limits quota
  - `cpu` and `memory` are limit-oriented quota keys in practice, not the same as `requests.cpu` / `requests.memory`

Verify:

```bash
kubectl describe limits -n <ns>
kubectl describe quota -n <ns>
```

## PriorityClass and Preemption

- `PriorityClass` affects scheduling and preemption, not QoS eviction.
- Preemption only works when the scheduler is involved.
- Do not use `nodeName` if you want to test preemption.
- Use `nodeSelector` or node affinity instead.

Gotchas:
- `nodeName` bypasses the scheduler, so no preemption happens.
- The high-priority pod must be unschedulable first, and evicting lower-priority pods must make enough room.

## Multi-Container Pods, InitContainers, and PDB

- `initContainer`: runs first and must complete before app containers start.
- Multi-container pod or sidecar: containers run together and share volume/network.
- `PodDisruptionBudget` protects against voluntary disruptions, not `kubectl delete pod`.

PDB quick check:

```bash
kubectl get pdb
kubectl describe pdb <name>
```

## Command-Line Gotchas

- `kubectl exec ... -- ls ... && cat ...`
- only the first command runs in the container
- `&&` is handled by your local shell unless wrapped with `sh -c`

Correct form:

```bash
kubectl exec -it cm-vol-pod -- sh -c 'cat /etc/settings/env.text; echo'
```

- `watch` inside `nginx` or `busybox` containers may fail because the binary is missing.
- Better:

```bash
watch 'kubectl exec cm-vol-pod -- cat /etc/settings/env.text'
```

or:

```bash
kubectl exec -it cm-vol-pod -- sh -c 'while true; do cat /etc/settings/env.text; echo; sleep 5; done'
```

## DaemonSet

Runs exactly **one pod per node**. Used for cluster-level agents (logging, monitoring, networking).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: monitoring
  updateStrategy:
    type: RollingUpdate        # or OnDelete
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule     # add this to run on control plane nodes too
      containers:
      - name: agent
        image: monitoring:1.0
```

```bash
kubectl get ds -n kube-system                     # list DaemonSets
kubectl rollout status daemonset/<name>
kubectl rollout undo daemonset/<name>
kubectl set image daemonset/<name> <container>=<image>
```

**Update strategies:**
- `RollingUpdate` (default): old pods replaced progressively
- `OnDelete`: new pods only created when old pods are manually deleted

> **Gotcha:** DaemonSets have no `replicas` field. Scale is controlled by node count.

> **Gotcha:** DaemonSets don't run on control plane nodes by default unless you add a toleration for the `node-role.kubernetes.io/control-plane:NoSchedule` taint.

---

## StatefulSet

Ordered, stable pod identity. Each pod gets a persistent name and its own PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web-headless"    # headless service name (required)
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 1Gi
```

**Pod naming:** `web-0`, `web-1`, `web-2` (ordinal, predictable)

**PVC naming:** `data-web-0`, `data-web-1`, `data-web-2`

**DNS per pod:** `web-0.web-headless.<ns>.svc.cluster.local`

> **Gotcha:** Deleting a StatefulSet does **not** delete PVCs — you must delete them manually.

> **Gotcha:** `serviceName` must reference a **headless** service (`clusterIP: None`).

---

## Jobs — Advanced Fields

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 6            # total successful completions needed
  parallelism: 2            # pods running simultaneously at any time
  backoffLimit: 3           # retries before job is marked Failed (default 6)
  activeDeadlineSeconds: 60 # job killed after this many seconds total
  template:
    spec:
      restartPolicy: OnFailure   # or Never (required for Jobs — cannot be Always)
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo done"]
```

| Field | Meaning |
|---|---|
| `completions` | N successful pod completions required for job success |
| `parallelism` | Max pods running at once |
| `backoffLimit` | Max pod retries before Job fails (exponential backoff: 10s, 20s, 40s...) |
| `activeDeadlineSeconds` | Hard timeout for entire job — overrides backoffLimit |

> **Gotcha:** `restartPolicy` must be `OnFailure` or `Never` for Jobs. `Always` is not allowed.

> **Gotcha:** Exit code 0 = success. Anything else = failure (counts against backoffLimit).

---

## CronJob — Advanced Fields

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"          # 2 AM daily (UTC)
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  startingDeadlineSeconds: 60     # skip if can't start within 60s of scheduled time
  suspend: false                  # true = pause all future runs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox
            command: ["sh", "-c", "echo running cleanup"]
```

**concurrencyPolicy:**
| Value | Behaviour |
|---|---|
| `Allow` (default) | New job runs even if previous job still running |
| `Forbid` | Skip new run if previous job still running |
| `Replace` | Cancel previous job and start the new one |

```bash
# Suspend/resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

# Trigger immediately (without waiting for schedule)
kubectl create job --from=cronjob/<name> manual-run-1
```

> **Gotcha:** If a CronJob misses 100 or more scheduled runs, it stops creating new jobs. Use `startingDeadlineSeconds` to control the window.

---

## Security Context

### Pod-level vs Container-level

```yaml
spec:
  securityContext:               # Pod-level — applies to ALL containers
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000               # group owner of mounted volumes
    runAsNonRoot: true
  containers:
  - name: app
    image: nginx
    securityContext:             # Container-level — overrides pod-level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

| Field | Level | Effect |
|---|---|---|
| `runAsUser` | Pod or Container | UID for process |
| `runAsGroup` | Pod or Container | GID for process |
| `fsGroup` | Pod only | GID for volume ownership |
| `runAsNonRoot` | Pod or Container | Reject if UID=0 |
| `allowPrivilegeEscalation` | Container only | Block setuid/setcap escalation |
| `readOnlyRootFilesystem` | Container only | Mount `/` as read-only |
| `capabilities.drop/add` | Container only | Linux capability control |

---

## Pod Lifecycle Hooks

```yaml
containers:
- name: app
  image: nginx
  lifecycle:
    postStart:
      exec:
        command: ["/bin/sh", "-c", "echo 'started' > /tmp/started"]
    preStop:
      exec:
        command: ["/bin/sh", "-c", "nginx -s quit; sleep 5"]
```

- **postStart**: Runs immediately after container start, in **parallel** with the main process. Container stays `ContainerCreating` until postStart completes.
- **preStop**: Runs **before** SIGTERM is sent. Use for graceful shutdown. Container waits for preStop to finish before receiving SIGTERM.

Handler types: `exec`, `httpGet`, `tcpSocket`

> **Gotcha:** If postStart fails, the container is killed and restarted.

---

## Rollout Controls

```bash
# Pause a rollout (changes accumulate but don't apply)
kubectl rollout pause deployment/<name>

# Resume the rollout
kubectl rollout resume deployment/<name>

# Watch live status
kubectl rollout status deployment/<name> -w

# Show revision history
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2   # details for one revision

# Undo last rollout
kubectl rollout undo deployment/<name>

# Undo to specific revision
kubectl rollout undo deployment/<name> --to-revision=2
```

> **Gotcha:** `kubectl rollout pause` + multiple `kubectl set image` / `kubectl set env` = batch changes applied together on `resume`. Efficient for multi-change rollouts.

---

## Inter-Pod Affinity and Anti-Affinity

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:    # hard
    - labelSelector:
        matchLabels:
          app: cache
      topologyKey: kubernetes.io/hostname   # "same node as a cache pod"
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:   # soft
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname  # "prefer not same node as another web pod"
```

**topologyKey options:**
- `kubernetes.io/hostname` — node level (most common)
- `topology.kubernetes.io/zone` — availability zone level

> **Gotcha:** Inter-pod affinity is expensive to compute at scale. Prefer topology spread constraints for spreading workloads evenly.

---

## Topology Spread Constraints

Distribute pods evenly across nodes or zones.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                        # max difference in pod count between any two domains
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule  # hard: don't schedule if violated
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 2
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway # soft: prefer spread but schedule anyway
    labelSelector:
      matchLabels:
        app: web
```

| `whenUnsatisfiable` | Behaviour |
|---|---|
| `DoNotSchedule` | Hard constraint — pod stays Pending if maxSkew would be violated |
| `ScheduleAnyway` | Soft constraint — schedule but try to minimize skew |

---

## envFrom — Bulk Environment Import

Instead of importing one key at a time, import all keys from a ConfigMap or Secret:

```yaml
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config      # ALL keys become env vars
    - secretRef:
        name: app-secret      # ALL keys become env vars
    env:
    - name: OVERRIDE_KEY      # individual entries override envFrom
      value: "override-value"
```

> **Gotcha:** If a ConfigMap key name is not a valid env var name (e.g., contains hyphens), the pod may fail to start. Use `env[].valueFrom.configMapKeyRef` for those.

---

## Deployment Recreate Strategy

```yaml
spec:
  strategy:
    type: Recreate            # kill ALL old pods first, then create new ones
```

Vs `RollingUpdate` (default): `Recreate` causes downtime but avoids running two versions simultaneously. Useful for databases that can't run two versions at once.

---

## Exam Habits

- Prefer explicit commands over clever shortcuts.
- Verify with `describe` when behavior matters.
- For Services: check both `svc` and `endpoints`.
- For Deployments: check `deploy`, `rs`, and `pods`.
- For rollout problems: use `kubectl describe pod`, `kubectl rollout status`, and `kubectl rollout history`.
- For Deployment problems: check `deploy`, `rs`, and `pods`, not only `pods`.
- If old and new ReplicaSet hashes are present at the same time, rollout overlap can hide the true scheduling math.
- `kubectl rollout undo` is faster than editing YAML when an image update breaks things.
- `kubectl create job --from=cronjob/<name> <job-name>` triggers a CronJob immediately.

---

## Workload & Scheduling Exam Gotchas

| Gotcha | Correct Approach |
|--------|---|
| DaemonSet running on wrong nodes | Check node taints; add tolerations to DS pod spec |
| DaemonSet won't run on control plane | Add toleration for `node-role.kubernetes.io/control-plane:NoSchedule` |
| StatefulSet PVCs not cleaned up | Must delete PVCs manually after deleting StatefulSet |
| Job restartPolicy: Always | Not allowed — use `OnFailure` or `Never` |
| CronJob runs overlap | Set `concurrencyPolicy: Forbid` to prevent overlapping runs |
| preStop not running long enough | Increase `terminationGracePeriodSeconds` (default 30s) |
| envFrom + invalid key name | Use `env[].valueFrom.configMapKeyRef` for hyphenated keys |
| Rollout pause forgot to resume | `kubectl rollout resume deployment/<name>` |
| nodeName bypasses scheduler | Use `nodeSelector` or affinity when testing preemption |

---

## ResourceQuota vs LimitRange

### ResourceQuota
Caps **total aggregate** consumption across the whole namespace.

```bash
k create quota rq -n myns --hard=requests.cpu=1000m,requests.memory=1Gi,limits.cpu=2,limits.memory=2Gi
```

Gotchas:
- `cpu` shorthand = `requests.cpu` (not limits)
- If quota exists for cpu/memory, every pod **must** specify requests/limits or it gets rejected
- Use LimitRange alongside to inject defaults automatically

Common hard fields:
- `requests.cpu`, `limits.cpu`, `requests.memory`, `limits.memory`
- `pods`, `services`, `secrets`, `configmaps`, `persistentvolumeclaims`
- `requests.storage`

Scopes — quota only applies to pods matching the scope:
- `BestEffort` / `NotBestEffort`
- `Terminating` / `NotTerminating`
- `PriorityClass` (via scopeSelector)

```yaml
scopeSelector:
  matchExpressions:
  - operator: In
    scopeName: PriorityClass
    values: ["high"]
```

### LimitRange
Sets **per-container/pod defaults and min/max bounds**. Namespaced.

| Field | Purpose |
|---|---|
| `default` | injected as limits if container doesn't specify |
| `defaultRequest` | injected as requests if container doesn't specify |
| `max` | upper bound — pod rejected if exceeded |
| `min` | lower bound — pod rejected if below |

Types:
- `Container` — applies per container (supports all 4 fields)
- `Pod` — applies to sum of all containers in pod (min/max only)
- `PersistentVolumeClaim` — applies per PVC (min/max only)

Mutation order: defaults injected first → then min/max validation.
| Inter-pod affinity wrong topologyKey | `kubernetes.io/hostname` = node, `topology.kubernetes.io/zone` = zone |
