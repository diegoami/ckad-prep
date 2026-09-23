# 082 — A Pod rejected by LimitRange min/max, fixed by sizing to half the ceiling

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

`043` shows the *default-filling* side of a `LimitRange` on a Pod with no `resources:` block.
This is the other half: a Pod that does specify resources, but outside the allowed min/max, gets
rejected outright rather than defaulted.

## Task

> Namespace `limitreject` has a `LimitRange` capping each container's CPU/memory between a min and
> a max. A Pod `worker` was rejected on creation — diagnose why, then fix it by setting its
> resources to exactly half the `LimitRange`'s maximum.

## Documentation

What to look up: **Limit Ranges** — the `min`/`max` gate, distinct from `default`/`defaultRequest`.
- <https://kubernetes.io/docs/concepts/policy/limit-range/> — same page as `043`, different section.

## Setup

```bash
kubectl create ns limitreject

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: limitreject
spec:
  limits:
  - type: Container
    min:
      cpu: 100m
      memory: 64Mi
    max:
      cpu: 800m
      memory: 640Mi
EOF
```

## Solution

Reproduce the rejection — a Pod requesting *above* the max:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: worker
  namespace: limitreject
spec:
  containers:
  - name: worker
    image: nginx:1.25-alpine
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 1
        memory: 1Gi
EOF
```
```
Error from server (Forbidden): pods "worker" is forbidden: [maximum cpu usage per Container is
800m, but limit is 1, maximum memory usage per Container is 640Mi, but limit is 1Gi]
```
Unlike a ResourceQuota rejection (`052`, `058`), this happens with **zero other Pods in the
namespace** — `LimitRange` caps each *individual* container against its own min/max, it has nothing
to do with cumulative usage across the namespace the way `ResourceQuota` does. The error also names
the exact ceiling, so there's no need to `kubectl describe limitrange` separately just to find it.

Read the `LimitRange`'s `max` directly and compute half of it — `800m`/`640Mi` → `400m`/`320Mi`:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: worker
  namespace: limitreject
spec:
  containers:
  - name: worker
    image: nginx:1.25-alpine
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 400m
        memory: 320Mi
EOF

kubectl wait --for=condition=Ready pod/worker -n limitreject --timeout=30s
# condition met
```

The *other* boundary rejects just as hard — a request **below** `min`:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tiny
  namespace: limitreject
spec:
  containers:
  - name: tiny
    image: nginx:1.25-alpine
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 400m
        memory: 320Mi
EOF
```
```
Error from server (Forbidden): pods "tiny" is forbidden: [minimum cpu usage per Container is
100m, but request is 50m, minimum memory usage per Container is 64Mi, but request is 32Mi]
```

**Lesson:** `LimitRange` is both a filler (`043`, when a field is *missing*) and a gate (here, when
a field is *present but out of range*) — the same object does both jobs depending on what the Pod
author did or didn't specify. "Half of the max" isn't a Kubernetes-enforced rule (same caveat as
`052`'s "double the requests") — it's just a safe, simple way to land comfortably inside an unknown
acceptable range without having to reason about the workload's real needs.

## Cleanup

```bash
kubectl delete ns limitreject
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
