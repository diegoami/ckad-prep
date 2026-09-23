# 024 — Drop all Linux capabilities except the ones a container actually needs

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Hard

## Task

> Run an nginx Pod (`cap-demo`, namespace `caps`, image `nginx:1.25-alpine`) that binds to a low
> port (80) but drops every Linux capability it doesn't strictly need, instead of running with the
> container runtime's full default capability set. Confirm it actually reaches `Running`, not just
> that it looks right on paper.

## Documentation

What to look up: **Configure a Security Context for a Pod or Container** — the capabilities section.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/> — the
  "Set capabilities for a Container" section covers `add`/`drop` under `securityContext.capabilities`.

## Setup

```bash
kubectl create ns caps
```

## Solution

### The naive answer is wrong

The obvious-looking answer is `drop: ["ALL"], add: ["NET_BIND_SERVICE"]` — NET_BIND_SERVICE is
what lets a process bind to a port below 1024. That alone isn't enough for `nginx:1.25-alpine`:
the master process also `chown`s its cache directories at startup, so without `CHOWN` (and
`SETGID`/`SETUID`, needed to drop from root to the `nginx` user after binding) it crashes
immediately with `chown(...) failed (1: Operation not permitted)`. Verified both ways on a live
cluster — this is exactly the kind of "looks right, fails at runtime" trap the exam rewards
catching before you move on to the next question.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cap-demo
  namespace: caps
spec:
  containers:
  - name: cap-demo
    image: nginx:1.25-alpine
    securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE", "CHOWN", "SETGID", "SETUID"]
EOF

kubectl -n caps wait --for=condition=ready pod/cap-demo --timeout=30s
kubectl -n caps get pod cap-demo
# READY 1/1
```

To see the trap for yourself: apply with just `add: ["NET_BIND_SERVICE"]` first, watch it fail
(`kubectl -n caps logs cap-demo`), then fix it — reproducing the debugging path is more
exam-realistic than starting from the correct answer.

## Cleanup

```bash
kubectl delete ns caps
```
