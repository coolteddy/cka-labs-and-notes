# CKA Notes: Workload and Scheduling

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

QoS classes:
- `BestEffort`: no requests or limits
- `Burstable`: some requests or limits set, but not all equal
- `Guaranteed`: every container has CPU and memory requests and limits, and requests equal limits

Check with:

```bash
kubectl describe pod <pod>
```

## LimitRange and ResourceQuota

- `LimitRange` sets namespace defaults, mins, and maxes for resources.
- `ResourceQuota` limits total namespace usage.
- `ResourceQuota` is cumulative, not per pod.

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

## Exam Habits

- Prefer explicit commands over clever shortcuts.
- Verify with `describe` when behavior matters.
- For Services: check both `svc` and `endpoints`.
- For Deployments: check `deploy`, `rs`, and `pods`.
- For rollout problems: use `kubectl describe pod`, `kubectl rollout status`, and `kubectl rollout history`.
