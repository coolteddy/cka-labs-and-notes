# CKA Notes: Storage

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
