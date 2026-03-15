# Troubleshooting Notes

This folder is for short Kubernetes troubleshooting writeups collected during CKA study.

Each writeup should capture:

- scenario
- symptoms
- commands used to diagnose it
- root cause
- fix
- takeaway

## Suggested Template

```md
# <short scenario title>

## Scenario

What was supposed to work, and what was broken.

## Symptoms

- `kubectl get ...`
- `kubectl describe ...`
- application behavior

## Diagnosis

- commands run
- what each command revealed

## Root Cause

Short explanation of the actual issue.

## Fix

What was changed to resolve it.

## Takeaway

What to remember for the CKA exam and for real cluster operations.
```

## Good Scenario Ideas

- Service has no endpoints
- DNS lookup fails because of namespace mismatch
- Ingress points to the wrong Service port
- NetworkPolicy blocks DNS or app traffic
- PVC stays pending
- Pod is running but not ready
