# CKA Notes: Storage

> **Domain weight: 10%.** Covers PV, PVC, StorageClass, volume types, and StatefulSet storage.

## Fast Commands

- Storage classes: `kubectl get sc`
- PVs and PVCs: `kubectl get pv,pvc -A`
- Describe a claim: `kubectl describe pvc <name> -n <ns>`
- Describe a volume: `kubectl describe pv <name>`
- Check pod mounts: `kubectl describe pod <pod> -n <ns>`
- Read from a mounted file: `kubectl exec -n <ns> <pod> -- cat /path/file`
- Explain storage fields:
  - `kubectl explain pv.spec`
  - `kubectl explain pvc.spec`
  - `kubectl explain pod.spec.volumes.persistentVolumeClaim`

## emptyDir

- `emptyDir` exists only for the lifetime of the Pod.
- Containers in the same Pod can share it.
- If the Pod is deleted and recreated, the data is lost.

Example shape:

```yaml
volumes:
- name: shared
  emptyDir: {}
```

## hostPath

- `hostPath` mounts a path from one node's filesystem.
- It is node-local, not shared across the cluster.
- Another Pod sees the same data only if it lands on the same node and mounts the same path.

Example shape:

```yaml
volumes:
- name: host
  hostPath:
    path: /data/test
    type: DirectoryOrCreate
```

## Static PV and PVC

- A static PV is created first.
- A PVC binds to a matching PV by size, access mode, storage class, and other constraints.
- If your cluster has a default `StorageClass`, set `storageClassName: ""` on the PVC when you want static binding without dynamic provisioning.

Static PVC example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static-1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""
```

## Dynamic Provisioning

- Dynamic provisioning uses a `StorageClass`.
- In this k3d cluster, `local-path` is the default class.
- `local-path` uses `WaitForFirstConsumer`, so a PVC can stay `Pending` until a Pod uses it.
- Once a consumer Pod exists, Kubernetes provisions a PV and binds the PVC.
- Deleting the Pod does not unbind the PVC. The claim stays `Bound`.

## Access Modes

- `ReadWriteOnce` means read-write mounted by a single node at a time.
- It does not mean "only one Pod ever".
- Multiple Pods can use the same `ReadWriteOnce` volume if they are on the same node and the storage plugin supports that behavior.

## Reclaim Policy

- `Delete`: underlying storage is removed automatically if supported.
- `Retain`: underlying storage is kept and manual cleanup is required.

Common PV states:

- `Available`: ready to bind
- `Bound`: attached to a PVC
- `Released`: claim was deleted, but the PV is not automatically reusable

For `Retain`, reuse usually means:

- clean the old data on the backend storage
- recreate or manually reset the PV for a new claim

## PVC Pending Troubleshooting

If a PVC is `Pending`, check:

1. `kubectl describe pvc <name> -n <ns>`
2. `kubectl get sc`
3. `kubectl get pv`

Common causes:

- bad `storageClassName`
- no matching static PV exists
- default dynamic provisioning was triggered when you wanted static binding
- `WaitForFirstConsumer`: no Pod is using the claim yet

Event examples:

- `storageclass.storage.k8s.io "<name>" not found`
- `no persistent volumes available for this claim and no storage class is set`

## Pod Proof Patterns

Use logs with clear markers:

```yaml
command:
- sh
- -c
- |
  echo "WRITE"
  echo "hello" > /data/file
  echo "READ"
  cat /data/file
  sleep 3600
```

Or keep the Pod simple and verify with `exec`:

```bash
kubectl exec -n <ns> <pod> -- cat /data/file
```

## Cheat Sheet

- `emptyDir`: Pod lifetime only
- `hostPath`: node-local path
- static PV/PVC: write YAML, often set PVC `storageClassName: ""`
- dynamic PVC: use `storageClassName`, often waits for first consumer
- `ReadWriteOnce`: one node, not one Pod
- `Retain`: data kept, manual cleanup required
- PVC `Pending`: check `describe pvc`, `get sc`, `get pv`
- PVC `Bound`: claim has a PV, even if no Pod is currently using it

## Exam Habits

- Scope everything with `-n <namespace>`.
- Remove generated noise such as `status: {}` from hand-edited manifests.
- For storage drills, verify with both `get` and `describe`.
- Use short, obvious file paths like `/data/file`, `/cache/check.txt`.
- Print explicit `WRITE` and `READ` markers in container commands.

---

## Complete Static PV + PVC YAML Reference

### PersistentVolume (all key fields)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain   # or Delete
  storageClassName: manual                # must match PVC storageClassName
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

### PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi        # PVC request must be ≤ PV capacity
  storageClassName: manual  # empty string "" = opt out of dynamic provisioning
```

### Static Binding — All Three Must Match
| Field | PV | PVC |
|---|---|---|
| `storageClassName` | `manual` | `manual` |
| `accessModes` | `[ReadWriteOnce]` | `[ReadWriteOnce]` |
| `storage` | `1Gi` (capacity) | `1Gi` (≤ PV) |

> **Gotcha:** `storageClassName: ""` (empty string) in PVC means "I want static binding, no dynamic provisioner". Omitting the field entirely may trigger dynamic provisioning if a default StorageClass exists.

> **Gotcha:** `storageClassName` is **immutable** on a PVC after creation. Delete and recreate if wrong.

---

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner    # for local/hostPath (static only)
reclaimPolicy: Retain                        # or Delete (default)
allowVolumeExpansion: true                   # allow PVC resize
volumeBindingMode: WaitForFirstConsumer      # or Immediate
```

### volumeBindingMode
| Mode | Behaviour |
|---|---|
| `Immediate` | PV provisioned/bound as soon as PVC is created |
| `WaitForFirstConsumer` | PV provisioned only when a Pod consumes the PVC — topology-aware |

### Volume Expansion (Resize PVC)
```bash
# Step 1: StorageClass must have allowVolumeExpansion: true
kubectl get sc <name> -o yaml | grep allowVolumeExpansion

# Step 2: Edit PVC to increase storage request
kubectl edit pvc <name> -n <ns>
# Change: resources.requests.storage: 2Gi  (must be LARGER, never smaller)

# Step 3: Verify
kubectl get pvc <name> -n <ns>
# Status.capacity will update once the resize completes
```

> **Gotcha:** You can only **increase** PVC size, never decrease. The StorageClass must have `allowVolumeExpansion: true`, otherwise the edit is rejected.

---

## hostPath Types Reference

```yaml
volumes:
- name: host-vol
  hostPath:
    path: /data/app
    type: DirectoryOrCreate    # creates dir if missing
```

| `type` | Behaviour |
|---|---|
| `""` (empty) | No checks performed |
| `DirectoryOrCreate` | Create directory if it doesn't exist |
| `Directory` | Must already exist |
| `FileOrCreate` | Create file if it doesn't exist |
| `File` | Must already exist |
| `Socket` | Must be a UNIX socket |

---

## subPath — Multiple Uses of One Volume

`subPath` lets multiple containers share one volume by mounting different subdirectories.

```yaml
spec:
  containers:
  - name: mysql
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: shared-data
      subPath: mysql          # maps to /mnt/data/mysql on the volume
  - name: nginx
    volumeMounts:
    - mountPath: /var/www/html
      name: shared-data
      subPath: html           # maps to /mnt/data/html on the volume
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: shared-pvc
```

Also used to mount a **single key from a ConfigMap as a file** (not the whole ConfigMap):
```yaml
volumeMounts:
- mountPath: /etc/config/app.conf
  name: config-vol
  subPath: app.conf           # only mount this key, not the whole ConfigMap
volumes:
- name: config-vol
  configMap:
    name: app-config
```

---

## StatefulSet — Storage with volumeClaimTemplates

Each StatefulSet pod gets its **own** PVC automatically. The PVCs persist even if the pod is deleted.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None           # required — headless service for StatefulSet DNS
  selector:
    app: db
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: "db-headless"   # must reference the headless service
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
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

**Resulting PVCs:** `data-db-0`, `data-db-1`, `data-db-2`

**Pod DNS:** `db-0.db-headless.<ns>.svc.cluster.local`

> **Gotcha:** Deleting a StatefulSet does **not** delete its PVCs. You must delete them manually.

> **Gotcha:** `serviceName` in the StatefulSet spec **must** reference a headless Service (`clusterIP: None`). Without it, ordered pod DNS won't work.

---

## Released PV — Fix for Rebinding

A PV enters `Released` state when its PVC is deleted. It can't bind to a new PVC until `claimRef` is cleared.

```bash
# Check PV status
kubectl get pv

# Clear claimRef to make Available again
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'

# Verify
kubectl get pv   # status should now be Available
```

---

## Storage Exam Gotchas

| Gotcha | Correct Approach |
|--------|---|
| PVC pending — silent default SC trigger | Set `storageClassName: ""` to force static binding |
| PVC `storageClassName` immutable | Delete and recreate PVC if wrong |
| Released PV won't bind | Clear claimRef: `kubectl patch pv <name> -p '{"spec":{"claimRef": null}}'` |
| Volume expansion fails | StorageClass must have `allowVolumeExpansion: true` |
| StatefulSet PVCs not deleted with StatefulSet | Must delete PVCs manually |
| RWO on multiple nodes | Multi-Attach error — RWO = one NODE, not one pod |
| StatefulSet without headless service | Pods won't get stable DNS identities |
| subPath and ConfigMap | Without subPath, mounting a ConfigMap as a volume replaces the whole directory |
