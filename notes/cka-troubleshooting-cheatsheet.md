# CKA Troubleshooting Cheatsheet

Covers the 30% domain: node failures, pod debugging, service issues, DNS, control plane, networking.

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

## General Troubleshooting Flow

```
kubectl get nodes                    # node health
kubectl get pods -A                  # cluster-wide pod status
kubectl describe <resource> <name>   # events + config
kubectl logs <pod> [--previous]      # app/container logs
journalctl -u kubelet -n 50         # node-level logs
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

### Log Commands
```bash
journalctl -u kubelet -n 50            # last 50 lines (stopped service)
journalctl -u kubelet -n 50 --no-pager # no pagination
journalctl -u kubelet -f               # live follow (running but failing)
journalctl -u kubelet --since "5m ago" # last 5 minutes
journalctl -u kubelet -n 100 | grep -i error  # filter errors
```

---

## 2. Pod Not Running

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
| CrashLoopBackOff | Container starts then exits | `kubectl logs --previous`, check command/args/env |
| ContainerCreating | Stuck setting up | CNI issue, volume mount issue, ConfigMap/Secret missing |
| ErrImagePull | First pull failure | Same as ImagePullBackOff |

### CrashLoopBackOff Debugging
```bash
kubectl logs <pod> --previous          # most reliable — logs from last crashed container
kubectl describe pod <pod>             # check command, args, env, probes
kubectl exec <pod> -- <cmd>            # if you can catch it running briefly
kubectl debug -it <pod> --image=busybox -- sh  # ephemeral container (shares network/PID namespace)
```

### Pending Pod — Common Causes
- **No node matches nodeSelector** → remove nodeSelector or label the node
- **Taint with no toleration** → add toleration or remove taint
- **Insufficient resources** → check requests vs allocatable
- **PVC not bound** → check PV exists with matching storageClassName and capacity
- **Scheduler down** → check `kube-scheduler` pod in `kube-system`

---

## 3. Service Not Working

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

## 4. DNS Issues

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

## 5. Control Plane Component Down

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

## 6. NetworkPolicy Blocking Traffic

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

## 7. Probes (Liveness/Readiness)

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

## 8. Cascading Failures

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

## 9. Kubernetes Component Flow

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

## 10. Drill Answers Summary

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

## Exam Gotchas

1. **Always specify `-n <namespace>`** — don't assume default
2. **`kubectl logs --previous`** — use for CrashLoopBackOff, not exec
3. **Endpoints empty ≠ no pods** — it means selector doesn't match
4. **Endpoints populated ≠ traffic works** — targetPort could be wrong
5. **After editing CoreDNS ConfigMap** — must restart CoreDNS deployment
6. **After editing static pod manifests** — kubelet auto-reconciles, but restart kubelet if stuck
7. **`kubeadm init phase`** — regenerate any control plane manifest without searching for files
8. **NodePort services** — accessible on ALL node IPs, also have a ClusterIP
9. **`/etc/default/kubelet`** — no daemon-reload needed; systemd unit files do need it
10. **Don't guess probe paths** — exec into pod or check logs to find valid endpoints
11. **API server port is always 6443** — memorise it
12. **Read the task carefully** — "fix without deleting" vs "fix by any means"
13. **Always verify end-to-end** — endpoints existing does not mean traffic is flowing
