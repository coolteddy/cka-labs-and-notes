# Exercises

This folder groups hands-on CKA practice files by topic.

## Layout

- `workloads/`: pods, deployments, replicasets, jobs, probes
- `scheduling/`: requests, limits, quotas, affinity, taints, tolerations, priority
- `services/`: service manifests and exposure exercises
- `crd/`: CRD and custom resource examples
- `networking/`: CNI, ingress, gateway, and related networking practice
- `platform/`: Helm and Kustomize exercises
- `scratch/`: one-off or unclassified files kept for reference

## Naming

- Keep topic files close to the exercise they belong to
- Use short task-oriented names such as `probe-web.yaml`, `pi-job.yaml`, `combo.yaml`
- Put temporary experiments in `scratch/` until they deserve a real topic folder
