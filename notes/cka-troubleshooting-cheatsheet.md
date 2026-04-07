# CKA Troubleshooting Cheatsheet

> **Domain weight: 30% — Highest single domain. Cluster Architecture + Troubleshooting = 55% combined.**
> Allowed docs during exam: `kubernetes.io/docs`, `helm.sh/docs`, `kubernetes.io/blog`

Covers: node failures, pod debugging, service issues, DNS, control plane, etcd, networking, storage, RBAC, scheduling.

---

## Important File Paths

### Control Plane (on master node)

| Path | Purpose |
|---|---|
| `/etc/kubernetes/manifests/` | Static pod manifests (apiserver, etcd, scheduler, controller-manager) |
| `/etc/kubernetes/admin.conf` | Admin kubeconfig |
| `/etc/kubernetes/controller-manager.conf` | Controller-manager kubeconfig |
| `/etc/kubernetes/scheduler.conf` | Scheduler kubeconfig |
| `/etc/kubernetes/pki/` | All cluster certificates and keys |
| `/etc/kubernetes/pki/ca.crt` | Cluster CA certificate |
| `/etc/kubernetes/pki/etcd/` | etcd certificates |
| `/var/lib/etcd/` | etcd data directory (default) |

### Kubelet (on all nodes)

| Path | Purpose | daemon-reload needed? |
|---|---|---|
| `/etc/default/kubelet` | Extra args (KUBELET_EXTRA_ARGS) | No |
| `/var/lib/kubelet/config.yaml` | Main kubelet config | No |
| `/etc/kubernetes/kubelet.conf` | Kubeconfig for kubelet → API server | No |
| `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` | Systemd unit overrides | **Yes** |

### CNI (on all nodes)

| Path | Purpose |
|---|---|
| `/etc/cni/net.d/` | CNI config files |
| `/opt/cni/bin/` | CNI plugin binaries |

### Kubeconfig (on host/admin machine)

| Path | Purpose |
|---|---|
| `~/.kube/config` | Default kubectl config |
| `$KUBECONFIG` | Override via environment variable |

---

## Default Ports

| Component | Port |
|---|---|
| API server | `:6443` |
| etcd client | `:2379` |
| etcd peer | `:2380` |
| kubelet | `:10250` |
| kube-scheduler | `:10259` |
| kube-controller-manager | `:10257` |

## Default Service IPs

| Component | ClusterIP |
|---|---|
| Kubernetes API service | `10.96.0.1` |
| CoreDNS (kube-dns) service | `10.96.0.10` |

---

## Universal Troubleshooting Flow

> **Always follow this sequence. Never jump straight to editing YAML.**

```bash
# 1. SET CONTEXT FIRST — wrong cluster = 0 marks
kubectl config use-context <cluster>
kubectl config set-context --current --namespace=<ns>

# 2. OBSERVE — what is broken?
kubectl get <resource> -n <ns>
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp

# 3. DESCRIBE — why is it broken?
kubectl describe <resource> <name> -n <ns>
# → Read "Events:" bottom-up (newest first)
# → Read "Conditions:" section

# 4. LOGS — what did the app say?
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous           # if already restarted
kubectl logs <pod> -n <ns> -c <container>       # multi-container pod

# 5. DIG DEEPER — node/system level?
ssh <node>
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -50
sudo crictl ps -a                               # all containers incl. exited
sudo crictl logs <container-id>

# 6. FIX — minimal change, right level

# 7. VERIFY — did it actually work?
kubectl get <resource> -w
kubectl exec <pod> -- <connectivity-test>
```

---

## kubectl describe node — Useful grep Keywords

```bash
kubectl describe node <name> | grep -i condition     # Ready, DiskPressure, MemoryPressure, PIDPressure
kubectl describe node <name> | grep -i taint         # taints applied to node
kubectl describe node <name> | grep -i unschedulable # cordoned?
kubectl describe node <name> | grep -A5 "Allocated"  # CPU/memory used vs allocatable
kubectl describe node <name> | grep -A5 "Capacity"   # total CPU/memory/pods capacity
kubectl describe node <name> | grep -i kubelet       # kubelet version
kubectl describe node <name> | grep -i event         # recent events
kubectl describe node <name> | grep -i pressure      # DiskPressure, MemoryPressure, PIDPressure
```

Quick combined check:
```bash
kubectl describe node <name> | grep -E "Taint|Condition|Pressure|Unschedulable|Allocated"
```

---

## 1. Node NotReady

### Diagnosis
```bash
kubectl describe node <name>         # check Conditions + Events
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 50          # or -f for live streaming
```

### Common Causes & Fixes

| Cause | Log Clue | Fix |
|---|---|---|
| kubelet stopped | `inactive (dead)` | `sudo systemctl start kubelet` |
| containerd stopped | `connection refused to containerd.sock` | `sudo systemctl start containerd` |
| Bad kubelet args | `--port=0` or similar | Edit `/etc/default/kubelet`, restart kubelet |
| Certificate expired | `certificate has expired` | `sudo kubeadm certs renew all` + restart kubelet |
| CNI not installed | `CNI plugin not found` | Reinstall CNI (Calico/Flannel) |
| Wrong API server URL | `connection refused` to wrong port | Fix `server:` in `/etc/kubernetes/kubelet.conf` |
| Wrong clusterDNS IP | `failed to sync...` | Fix `clusterDNS` in `/var/lib/kubelet/config.yaml` |
| cgroupDriver mismatch | `failed to create cpuset cgroup` | Match `cgroupDriver` to container runtime (systemd) |

### Kubelet Config Files to Check

```bash
# Main config — clusterDNS, staticPodPath, cgroupDriver, eviction settings
sudo cat /var/lib/kubelet/config.yaml | grep -E "clusterDNS|staticPodPath|cgroupDriver"

# Kubeconfig — API server URL and port (must be https://<cp>:6443)
sudo cat /etc/kubernetes/kubelet.conf | grep server

# Systemd unit overrides — extra args, if any
sudo cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

> **Gotcha:** After editing any kubelet config file, always run:
> `sudo systemctl daemon-reload && sudo systemctl restart kubelet`
> Config changes don't apply until restart.

### Log Commands
```bash
journalctl -u kubelet -n 50            # last 50 lines (stopped service)
journalctl -u kubelet -n 50 --no-pager # no pagination
journalctl -u kubelet -f               # live follow (running but failing)
journalctl -u kubelet --since "5m ago" # last 5 minutes
journalctl -u kubelet -n 100 | grep -i error  # filter errors
```

---

## 2. DiskPressure

### What it is
When disk usage crosses a threshold, kubelet sets `DiskPressure=True` on the node and automatically adds a taint:
```
node.kubernetes.io/disk-pressure:NoSchedule
```
No new pods are scheduled on the node until resolved.

### Diagnosis
```bash
kubectl describe node <name> | grep -E "DiskPressure|Taint"

# On the node:
df -h                            # overall disk usage per partition
du -sh /var/lib/containerd/      # container images/layers
du -sh /var/log/                 # system/app logs
du -sh /var/lib/kubelet/         # pod volumes, emptyDir
du -sh /var/lib/etcd/            # etcd data (master only)
```

### Fix
```bash
# Clean unused container images
crictl rmi --prune

# Clean up stopped containers
crictl rm $(crictl ps -a -q --state exited)

# Trim journal logs
sudo journalctl --vacuum-size=100M
sudo journalctl --vacuum-time=2d
```

Once disk usage drops below the threshold, kubelet **automatically** clears DiskPressure — no manual intervention needed.

### Default eviction thresholds (`/var/lib/kubelet/config.yaml`)
```yaml
evictionHard:
  nodefs.available: "10%"     # trigger below 10% free
  nodefs.inodesFree: "5%"     # inode pressure
  imagefs.available: "15%"    # image filesystem
```

---

## 3. Resource Constraint (Pod Stuck Pending)

### Diagnosis
```bash
kubectl describe pod <name>     # Events: "0/2 nodes are available: insufficient cpu"
kubectl describe nodes | grep -A5 "Allocated resources"   # see what's used vs available
```

### Fix
Reduce the pod's resource requests to fit within available node capacity:
```bash
kubectl edit pod <name>    # reduce resources.requests.cpu / resources.requests.memory
```

### Common mistake
Don't confuse `requests` with `limits`:
- `requests` — what the scheduler uses to find a node (affects scheduling)
- `limits` — what the container is allowed to use at runtime (affects throttling/OOM)

A pod stuck Pending is always a **requests** problem, not limits.

---

## 4. Pod Not Running

### Quick Diagnosis
```bash
kubectl get pods -n <ns>
kubectl describe pod <name> -n <ns>    # check Events section
kubectl logs <pod> -n <ns> [--previous]
```

### Pod Status Cheat Table

| Status | Meaning | First Check |
|---|---|---|
| Pending | Not scheduled | `kubectl describe pod` → Events (scheduler, taints, resources, nodeSelector) |
| ImagePullBackOff | Wrong image or no access | Check image name, tag, registry credentials |
| ErrImagePull | First pull failure (becomes ImagePullBackOff) | Same as ImagePullBackOff |
| CrashLoopBackOff | Container starts then exits repeatedly | `kubectl logs --previous`, check exit code |
| ContainerCreating | Stuck setting up | CNI issue, volume mount issue, ConfigMap/Secret missing |
| Init:0/1 | Init container failing | `kubectl logs <pod> -c <init-container-name>` |
| OOMKilled | Memory limit exceeded | Increase `resources.limits.memory` |
| Running, 0/1 Ready | Readiness probe failing | `kubectl describe pod` → probe config + Events |
| Terminating (stuck) | Node dead or finalizer blocking | Force delete if node confirmed dead |
| Completed | Job/init container done | Normal — not an error |

### CrashLoopBackOff Debugging

```bash
kubectl logs <pod> --previous          # most reliable — logs from last crashed container
kubectl describe pod <pod>             # check command, args, env, probes
kubectl exec <pod> -- <cmd>            # if you can catch it running briefly
kubectl debug -it <pod> --image=busybox -- sh  # ephemeral container (shares network/PID namespace)
```

### Exit Code Reference (from `kubectl describe pod` → Last State)

| Exit Code | Meaning | Usual Cause |
|---|---|---|
| 0 | Clean exit | App finished (not a crash — check `restartPolicy`) |
| 1 | General app error | App-level bug, bad config, missing env/secret |
| 137 | OOMKilled (SIGKILL) | Memory limit exceeded |
| 139 | Segmentation fault | Binary/runtime crash |
| 143 | Graceful shutdown (SIGTERM) | Liveness probe killed it |

```bash
# Read exit code
kubectl describe pod <name> | grep -A5 "Last State"
```

### OOMKilled Fix
```bash
kubectl describe pod <name> | grep -E "OOMKilled|Last State"
# Fix: increase memory limit in the pod spec
kubectl edit deployment <name>
# Update: resources.limits.memory: "512Mi"  → "1Gi"
```

> **Gotcha:** OOMKilled happens when the container hits its `limits.memory`, even if the **node** has plenty of free RAM. The limit is per-container, enforced regardless of node capacity.

### Pending Pod — Common Causes
- **No node matches nodeSelector** → remove nodeSelector or label the node
- **Taint with no toleration** → add toleration or remove taint
- **Insufficient resources** → check requests vs allocatable
- **PVC not bound** → check PV exists with matching storageClassName and capacity
- **Scheduler down** → check `kube-scheduler` pod in `kube-system`

---

## 5. Service Not Working

### Diagnosis Flow
```bash
kubectl get svc <name> -n <ns>
kubectl describe svc <name> -n <ns>    # check Endpoints
kubectl get pods --show-labels -n <ns> # check pod labels
```

### Common Causes

| Symptom | Cause | Fix |
|---|---|---|
| Endpoints empty | Selector mismatch | Fix `spec.selector` to match pod labels |
| Endpoints populated but no response | Wrong targetPort | Fix `targetPort` to match container port |
| Connection refused | No pods running | Fix the pods first |

### Verify End-to-End
```bash
# Check container port
kubectl get pods -o jsonpath='{.items[0].spec.containers[0].ports[*].containerPort}'

# Check service targetPort
kubectl get svc <name> -o jsonpath='{.spec.ports[0].targetPort}'

# Test from inside cluster (works for ANY service type)
kubectl run test --image=busybox --rm -it -- wget -qO- http://<svc-name>.<namespace>/
```

### Service Types

| Type | ClusterIP | Inside cluster | Outside cluster |
|---|---|---|---|
| ClusterIP | Yes | Yes | No |
| NodePort | Yes | Yes | Yes (any node IP + port) |
| LoadBalancer | Yes | Yes | Yes (external LB) |
| Headless (clusterIP: None) | No | DNS only (returns pod IPs) | No |
| ExternalName | No | DNS redirect to external domain | No |

NodePort/LoadBalancer **include** a ClusterIP — they're layered. You can always test a NodePort service from inside the cluster using its ClusterIP or DNS name.

---

## 6. DNS Issues

### Diagnosis
```bash
kubectl exec <pod> -- nslookup kubernetes.default
kubectl exec <pod> -- nslookup google.com
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Internal DNS Fails (kubernetes.default)
- CoreDNS pods not running → check CoreDNS deployment
- kube-proxy missing iptables rules → `kubectl rollout restart ds/kube-proxy -n kube-system`

### External DNS Fails (google.com) but Internal Works
- CoreDNS `forward` pointing to wrong upstream
```bash
kubectl get configmap coredns -n kube-system -o yaml   # check forward . <ip>
# Fix: change to forward . /etc/resolv.conf  or  forward . 8.8.8.8
kubectl rollout restart deployment/coredns -n kube-system   # MUST restart after ConfigMap change
```

---

## 7. Control Plane Component Down

### Diagnosis
```bash
kubectl get pods -n kube-system
ls /etc/kubernetes/manifests/         # static pod manifests
crictl ps -a                          # if kubectl not working
```

### Common Causes
- Manifest missing from `/etc/kubernetes/manifests/`
- Wrong image name/tag in manifest
- Wrong command or arguments
- Wrong volume mounts (e.g. etcd data-dir)

### Regenerate Static Pod Manifests (exam-safe, no need to find lost files)
```bash
sudo kubeadm init phase control-plane apiserver
sudo kubeadm init phase control-plane controller-manager
sudo kubeadm init phase control-plane scheduler
sudo kubeadm init phase control-plane all    # all three at once
sudo kubeadm init phase etcd local           # etcd
```

Reference: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/

### After Editing Static Pod Manifests
- kubelet watches `/etc/kubernetes/manifests/` and auto-restarts pods
- If pod doesn't restart: `sudo systemctl restart kubelet`

---

## 8. NetworkPolicy Blocking Traffic

### Diagnosis
```bash
kubectl get networkpolicy -n <ns>
kubectl describe networkpolicy <name> -n <ns>
```

### Key Concepts
- **No NetworkPolicy** = all traffic allowed
- **Any NetworkPolicy selecting a pod** = all traffic denied except what's explicitly allowed
- Check both `policyTypes` (Ingress/Egress) and the rules

### Fix Approaches
- Add correct label to the source/destination pod to match the policy's `from`/`to` selector
- Update the NetworkPolicy to allow the required traffic
- Check namespace labels if using `namespaceSelector`

---

## 9. Probes (Liveness/Readiness)

### Diagnosis
```bash
kubectl describe pod <name>            # check probe config + Events
kubectl logs <pod> --previous          # 404s or connection errors in access log
```

### Debugging When Pod Keeps Crashing
```bash
kubectl logs <pod> --previous                    # most reliable — no timing needed
kubectl exec <pod> -- curl -s localhost:<port>/   # if you can catch it running
kubectl debug -it <pod> --image=busybox -- sh     # ephemeral container
```

### kubectl debug Notes
- Shares network namespace with target pod (localhost reaches the app)
- Shares PID namespace (can see app processes)
- Temporary — disappears when you exit
- Injected into the target pod, not a separate pod
- Check: `kubectl get pod <name> -o jsonpath='{.spec.ephemeralContainers}'`

---

## 10. Cascading Failures

### Example: Missing kube-proxy iptables rule
```
kube-proxy missing iptables rule for 10.96.0.1
  → Calico CNI can't reach API server via ClusterIP
    → CNI can't set up pod network namespace
      → kubelet can't create pod sandbox
        → ALL new pods fail to start
```

**Fix:** `kubectl rollout restart daemonset/kube-proxy -n kube-system`

**Lesson:** One missing iptables rule can break pod creation cluster-wide.

---

## 11. Kubernetes Component Flow

### Deployment Creation Flow
```
kubectl create deployment
  → API Server (validates, stores)
    → etcd (persists)
      → Controller Manager (creates ReplicaSet → creates Pod objects)
        → Scheduler (assigns Pod to Node)
          → kubelet (calls CRI → containerd → CNI → Calico)
            → kubelet reports Pod status → API Server → etcd
              → kube-proxy (updates iptables if Service exists)
```

### All Components Watch API Server (no polling)

| Component | Watches for |
|---|---|
| Controller Manager | Deployments, ReplicaSets, Pods, Nodes |
| Scheduler | Pods with no nodeName |
| kubelet | Pods bound to its node |
| kube-proxy | Services and Endpoints |

### Network Responsibility

| Responsibility | Owner |
|---|---|
| Service ClusterIP → Pod routing (iptables) | kube-proxy |
| Pod IP assignment | CNI (Calico) |
| Pod network namespace setup | kubelet → CRI → CNI |
| Pod-to-pod routing across nodes | CNI (Calico) |

---

## 12. Drill Answers Summary

| # | Scenario | Root Cause | Fix |
|---|---|---|---|
| 0 | kube-proxy iptables missing | kube-proxy lost sync | `kubectl rollout restart ds/kube-proxy -n kube-system` |
| 1 | Service not routing | Wrong selector `app: broken` | `kubectl edit svc` → fix selector to match pod labels |
| 2 | Pod not starting | Missing ConfigMap | `kubectl create configmap app-config --from-literal=setting=value` |
| 3 | Pod keeps restarting | Container exits (`exit 1`) | Fix command to stay alive (e.g. `sleep infinity`) |
| 4 | Static pod ImagePullBackOff | Typo in image `kube-schedulerr` | Fix image in manifest or `kubeadm init phase control-plane scheduler` |
| 5 | Pods stuck Pending | nodeSelector + taint, no matching node | Remove nodeSelector and/or taint |
| 6 | External DNS failing | CoreDNS forward to invalid IP `1.2.3.4` | Fix ConfigMap forward directive + restart CoreDNS |
| 7 | NetworkPolicy blocking | Ingress only allows `app: allowed` | Add matching label to client pod |
| 8 | kubectl failing | Wrong port in kubeconfig (9999 vs 6443) | Fix server port in `~/.kube/config` |
| 9 | Controller-manager missing | Manifest moved out of manifests dir | `sudo kubeadm init phase control-plane controller-manager` |
| 10 | Multi-issue: worker + app | kubelet --port=0, wrong image, wrong targetPort | Fix `/etc/default/kubelet`, fix image, fix targetPort |

---

## 13. etcd — Backup, Restore, and Health

### Check etcd Health
```bash
# etcd runs as a static pod — check it first
kubectl get pods -n kube-system | grep etcd
kubectl logs -n kube-system etcd-<node>

# Direct health check on control plane node
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# List members
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

> **Tip:** Get cert paths from the etcd static pod manifest:
> `grep -E "cacert|cert|key" /etc/kubernetes/manifests/etcd.yaml`

### Backup
```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd-snapshot.db --write-out=table
```

### Restore — THE TRIPLE UPDATE RULE

> **⚠️ Most common etcd restore failure:** Updating only ONE of the three required locations in the manifest.

```bash
# Step 1: Restore to a NEW directory
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Step 2: Edit the static pod manifest — THREE places must all match the new path
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

The three fields to update (all must point to `/var/lib/etcd-restore`):

```yaml
# 1. The --data-dir flag (in command: section)
- --data-dir=/var/lib/etcd-restore       # was: /var/lib/etcd

# 2. The volumeMount inside the etcd container
volumeMounts:
- mountPath: /var/lib/etcd-restore       # was: /var/lib/etcd
  name: etcd-data

# 3. The hostPath volume (what gets mounted from the node)
volumes:
- hostPath:
    path: /var/lib/etcd-restore          # was: /var/lib/etcd
    type: DirectoryOrCreate
  name: etcd-data
```

```bash
# Step 3: kubelet auto-restarts etcd (~30s). Watch it recover:
sudo crictl ps -a | grep etcd
kubectl get pods -n kube-system | grep etcd
kubectl get nodes   # verify cluster accessible
```

---

## 14. Certificate Issues

### Check Expiry
```bash
sudo kubeadm certs check-expiration   # all control plane certs

# Check a specific cert manually
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 "Validity"

# Check kubelet client cert on a worker node
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout | grep -A2 "Validity"
```

### Renew Certificates
```bash
sudo kubeadm certs renew all        # renew everything
sudo kubeadm certs renew apiserver  # renew specific cert

# CRITICAL: restart kubelet after renewal — new certs are on disk but
# control plane components still hold the OLD certs in memory
sudo systemctl restart kubelet

# kubectl may also fail after renewal — update your kubeconfig
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```

> **Gotcha:** `kubeadm certs renew` writes new certs to disk but doesn't reload components. Without restarting kubelet, you'll still get `x509: certificate has expired` errors.

---

## 15. Control Plane Static Pod — Deep Debugging

When `kubectl` doesn't work (API server down), use `crictl` directly on the control plane node.

### crictl Commands
```bash
sudo crictl ps -a                          # all containers incl. exited ones
sudo crictl ps -a | grep apiserver         # find the apiserver container
sudo crictl logs <container-id>            # read container logs
sudo crictl inspect <container-id>         # full container config JSON
```

### Broken Manifest — Common Exam-Planted Bugs

| Bug | Where | Symptom |
|---|---|---|
| Wrong `--etcd-servers` port (`2380` instead of `2379`) | kube-apiserver manifest | API server starts then crashes |
| Wrong cert file path (typo) | kube-apiserver or etcd manifest | `no such file or directory` in logs |
| Wrong `--advertise-address` | kube-apiserver manifest | clients can't reach API server |
| Duplicate or malformed flag | any manifest `command:` section | container exits immediately |
| Wrong image name (typo) | any manifest | ImagePullBackOff |
| Wrong `--kubeconfig` path | scheduler or controller-manager | component can't auth to API server |

```bash
# Workflow
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/   # backup first!
sudo crictl logs <id>    # read the exact error
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml         # fix it
# Wait 15-30s for kubelet to detect change and restart the pod
sudo crictl ps -a | grep apiserver   # watch for Running
kubectl get nodes                    # verify
```

> **Gotcha:** kubelet takes 15–30s to detect a manifest change. Be patient — don't re-edit thinking it didn't apply.

---

## 16. Scheduling — Taint & Affinity Failures

### Taint Effects Reference

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New pods won't be scheduled (existing pods unaffected) |
| `PreferNoSchedule` | Scheduler tries to avoid the node but will use it if needed |
| `NoExecute` | New pods won't schedule AND existing pods without toleration are evicted |

```bash
kubectl describe node <node> | grep Taint
kubectl taint nodes <node> <key>=<value>:<effect>        # add
kubectl taint nodes <node> <key>=<value>:<effect>-       # remove (trailing -)
```

### nodeAffinity Rules

| Rule | Behaviour |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard rule — pod won't schedule if no match |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft rule — prefers but schedules elsewhere if needed |

```bash
# Check node labels
kubectl get nodes --show-labels
kubectl label nodes <node> <key>=<value>   # add missing label
```

> **Gotcha:** A pod with `requiredDuringScheduling...` affinity that matches no node will stay Pending forever — even if the node has resources. Fix the label on the node or loosen the affinity.

---

## 17. Storage Failures

### PVC Stuck Pending

```bash
kubectl describe pvc <name> -n <ns>   # read Events section
kubectl get pv                        # check available PVs
kubectl get sc                        # check StorageClasses
```

| PVC Pending Reason | Root Cause | Fix |
|---|---|---|
| `no persistent volumes available` | No matching static PV | Create a PV matching PVC spec exactly |
| `storageclass not found` | Wrong `storageClassName` | Fix the name (or delete+recreate PVC — `storageClassName` is immutable) |
| `AccessModes mismatch` | PVC wants RWX, PV only has RWO | Fix access modes to match |
| `WaitForFirstConsumer` | Topology-aware SC, waiting for pod | Create the consuming pod to trigger binding |
| Volume in Released state | PV was bound to deleted PVC | Clear claimRef — see below |

```bash
# Clear a Released PV so it can bind again
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

### Static PV Binding Rules — ALL must match
- `storageClassName` (exact string, or both empty `""`)
- `accessModes` (PVC modes must be a subset of PV modes)
- `storage` (PVC request ≤ PV capacity)

> **Gotcha:** `storageClassName` is **immutable** in PVC after creation. If it's wrong, delete and recreate the PVC.

### ReadWriteOnce Contention
```bash
kubectl describe pod <name>
# Events: Multi-Attach error — Volume already exclusively attached to one node
```
RWO volumes can only be mounted on **one node** at a time. Multiple pods on different nodes → second pod fails.

Fix: ensure all pods using the RWO volume schedule to the same node, or use `ReadWriteMany` (requires NFS/Ceph).

---

## 18. RBAC & Access Failures

### "Forbidden" Error — Diagnosis
```bash
# Error: User "jane" cannot list resource "pods" in namespace "default"

# Test permissions
kubectl auth can-i list pods --as=jane
kubectl auth can-i list pods --as=jane -n production
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa

# List ALL permissions for a user
kubectl auth can-i --list --as=jane -n <ns>

# Inspect bindings
kubectl get role,rolebinding,clusterrole,clusterrolebinding -n <ns>
kubectl describe rolebinding <name> -n <ns>
```

### Create Role + Bind
```bash
# Role (namespace-scoped)
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n <ns>
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=<ns>:<sa-name> -n <ns>

# ClusterRole (cluster-scoped)
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=jane
```

> **Gotcha:** When binding to a ServiceAccount use `--serviceaccount=<namespace>:<name>`, NOT `--user=<sa-name>`. Those create different binding subjects.

---

## 19. Resource Quota Blocking Pods

```bash
# Error: "pods" is forbidden: exceeded quota: ...

kubectl describe resourcequota -n <ns>
# Shows Used / Hard columns for each resource

kubectl describe limitrange -n <ns>
# Shows default requests/limits applied if pod doesn't specify them
```

> **Gotcha:** If a namespace has a ResourceQuota on CPU/memory, **every pod must declare `resources.requests` and `resources.limits`**. Pods without them are rejected immediately. This is the #1 "why won't my pod create" surprise when quotas are active.

```yaml
# Required when ResourceQuota is active in the namespace
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

---

## 20. kubectl top Not Working (Metrics Server)

```bash
kubectl top nodes
# Error: metrics not available yet

# Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system <metrics-server-pod>

# Common fix — needs --kubelet-insecure-tls in kubeadm clusters
kubectl edit deployment metrics-server -n kube-system
# Add to args:
# - --kubelet-insecure-tls
```

> **Gotcha:** In kubeadm clusters, kubelet uses self-signed certs. Without `--kubelet-insecure-tls`, metrics-server can't scrape data and `kubectl top` always errors. This flag is needed in most exam cluster setups.

---

## 21. kube-proxy Failures

kube-proxy runs as a **DaemonSet**. If broken, Services won't route traffic even if pods are healthy.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# Force restart all kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system

# Check the kube-proxy ConfigMap for misconfiguration
kubectl get cm kube-proxy -n kube-system -o yaml
# Look at: clusterCIDR, mode (iptables vs ipvs)
```

---

## 22. Exam Scenario Catalogue

High-frequency tasks from candidate reports. Each takes 5–12 minutes.

| Scenario | Symptom | Root Cause Pattern | Key Fix |
|---|---|---|---|
| **A: Broken API server static pod** | `kubectl` hangs/refused | Wrong flag or path in manifest | `crictl logs` → fix manifest → wait 30s |
| **B: Worker node NotReady** | Node shows NotReady | kubelet config bug | `journalctl -u kubelet` → fix config → daemon-reload + restart |
| **C: etcd backup** | Task asks for snapshot | — | `etcdctl snapshot save` with all 3 cert flags |
| **D: etcd restore** | Task asks to restore | — | `etcdctl snapshot restore` → update ALL 3 manifest locations |
| **E: CrashLoopBackOff app** | Pod restarts constantly | Wrong env/cmd/probe | `logs --previous` → identify → edit deployment |
| **F: Service no endpoints** | Connectivity failure | Selector mismatch | Compare `svc.selector` vs `pod.labels` → fix |
| **G: NetworkPolicy cross-namespace** | Pod can't reach another NS | Missing `namespaceSelector` | Add `namespaceSelector` with namespace label |
| **H: PVC stuck Pending** | Pod stuck ContainerCreating | No matching PV, wrong SC, wrong mode | `describe pvc` → fix or create PV |
| **I: Deployment rollout broken** | New pods all failing | Bad image after update | `kubectl rollout undo deployment <name>` |
| **J: Cert expiry** | `x509: certificate has expired` | Certs expired | `kubeadm certs renew all` → restart kubelet → update kubeconfig |
| **K: Scheduling impossible** | Pod forever Pending | Taint + no toleration, or wrong nodeAffinity | Describe pod → fix toleration or label node |
| **L: kubectl top fails** | metrics not available | metrics-server missing `--kubelet-insecure-tls` | Edit metrics-server deployment |

---

## Exam Gotchas

1. **Always `kubectl config use-context`** — wrong cluster = 0 marks even with correct fix
2. **Always specify `-n <namespace>`** — don't assume default
3. **`kubectl logs --previous`** — use for CrashLoopBackOff, not exec
4. **Endpoints empty ≠ no pods** — it means selector doesn't match pod labels
5. **Endpoints populated ≠ traffic works** — targetPort could be wrong
6. **After editing CoreDNS ConfigMap** — must restart CoreDNS deployment
7. **After editing static pod manifests** — kubelet auto-reconciles in 15-30s, restart kubelet if stuck
8. **`kubeadm init phase`** — regenerate any control plane manifest without searching for files
9. **NodePort services** — accessible on ALL node IPs, also have a ClusterIP
10. **`/etc/default/kubelet`** — no daemon-reload needed; systemd unit files do need it
11. **Don't guess probe paths** — exec into pod or check logs to find valid endpoints
12. **API server port is always 6443** — memorise it
13. **Read the task carefully** — "fix without deleting" vs "fix by any means"
14. **Always verify end-to-end** — endpoints existing does not mean traffic is flowing
15. **etcd restore: ALL THREE locations** — `--data-dir`, `mountPath`, `hostPath.path` must all match
16. **After `kubeadm certs renew`** — restart kubelet AND refresh `~/.kube/config`
17. **OOMKilled ≠ node out of memory** — container hit its own `limits.memory` ceiling
18. **`crictl` not `docker`** — Docker removed from modern k8s; use `crictl ps/logs/inspect`
19. **`namespaceSelector` uses labels, not names** — use `kubernetes.io/metadata.name: <ns>` label
20. **PVC `storageClassName` is immutable** — delete and recreate PVC if wrong, you can't edit it
21. **ResourceQuota active → all pods need requests + limits** — forgetting this = instant pod rejection
22. **Released PV ≠ Available** — must clear `claimRef` before it can rebind
23. **SA binding syntax** — `--serviceaccount=<ns>:<name>` not `--user=<sa-name>`
24. **Static pod backup** — always `cp /etc/kubernetes/manifests/<name>.yaml /tmp/` before editing
