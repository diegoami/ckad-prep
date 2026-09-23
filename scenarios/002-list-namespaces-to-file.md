# 002 — List namespaces to a file

**Domain:** Core kubectl · **Difficulty:** Easy

## Task

> Someone on the platform team is auditing how the cluster is carved up and wants a snapshot of
> every Namespace that currently exists. Save that list to `~/ckad/002/namespaces`. It's fine if
> the file also includes extra columns such as STATUS or AGE.

## Documentation

What to look up: **Namespaces**.
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/> — what a
  Namespace is, and (via `kubectl get -h`) the `-o name`/output-format flags for redirecting to a file.

## Setup

Nothing to provision — every cluster has namespaces already. Just create the output directory.

```bash
mkdir -p ~/ckad/002
```

## Solution

```bash
kubectl get ns > ~/ckad/002/namespaces
cat ~/ckad/002/namespaces
# NAME                 STATUS   AGE
# default              Active   ...
# kube-node-lease      Active   ...
# ...
```

Plain shell redirection is all it takes. Two things to watch on the exam: write to exactly the path
given (tasks like this are graded by reading the file), and make sure you're on the right cluster
context before running anything — the exam gives you the `kubectl config use-context` command at
the top of each question.

## Cleanup

```bash
rm -rf ~/ckad/002
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
