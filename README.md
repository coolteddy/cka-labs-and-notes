# Kubernetes CKA Study Repo

Hands-on Kubernetes study material focused on CKA-style tasks, verification, and troubleshooting.

This repo is used for:

- topic notes
- drill manifests
- troubleshooting writeups
- a visible study history over time

## Structure

- `notes/`: compact topic notes and cheat sheets
- `exercises/`: manifests and hands-on lab material by topic
- `troubleshooting/`: scenario writeups with symptoms, diagnosis, fix, and takeaway

## Topics Covered

- workloads and scheduling
- storage
- networking
- troubleshooting

More topics will be added as study progresses.

## Study Approach

The focus here is not just writing YAML. Each topic is practiced using a drill-and-review flow:

1. Create or fix the Kubernetes objects by hand
2. Verify behavior with `kubectl get`, `describe`, `logs`, and `exec`
3. Capture failure patterns and root causes
4. Summarize key exam habits and cheat-sheet notes

## Public Repo Rules

This repository is intended to stay public, so it should not contain:

- kubeconfig files
- tokens, certificates, or private keys
- machine-specific shell history
- sensitive cluster output that should not be public

## Usage

Open the repo and work through the notes and exercises by topic. For realistic practice, run the manifests against a local lab cluster and keep troubleshooting notes for anything that fails unexpectedly.

## Progress

Current note sets:

- [notes/cka-shell-vi-tips.md](notes/cka-shell-vi-tips.md)
- [notes/cka-workload-scheduling-notes.md](notes/cka-workload-scheduling-notes.md)
- [notes/cka-storage-notes.md](notes/cka-storage-notes.md)
- [notes/cka-networking-notes.md](notes/cka-networking-notes.md)
