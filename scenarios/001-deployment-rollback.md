# 001 — Roll back a broken Deployment

**Domain:** Application Deployment · **Difficulty:** Medium

## Task

> The `heron` team runs the Deployment `orders-api` in Namespace `heron`. Earlier today someone
> shipped a change to it, and the Pods from that change have never become available. Use the
> Deployment's rollout history to get `orders-api` back onto the most recent revision that was
> actually healthy. Afterwards, be ready to tell the team what the bad change was and why it failed.

## Documentation

What to look up: **Deployments** — rollout history and rollback.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment> —
  the exact `kubectl rollout undo`/`kubectl rollout history` syntax and what a "revision" actually is.

## Setup

On the real exam the broken Deployment would already be waiting for you. On your local kind
cluster, build that state first: a healthy revision 1, then a revision 2 that can never finish
rolling out because its image tag has a typo.

```bash
kubectl create ns heron

# revision 1 — healthy
kubectl -n heron create deployment orders-api --image=nginx:1.25-alpine --replicas=2
kubectl -n heron rollout status deployment orders-api --timeout=90s

# revision 2 — the change that never came online
kubectl -n heron set image deployment/orders-api nginx=nginx:1.255-alpine
```

Don't wait for the second rollout to finish — it won't. That's the starting state; the task
begins here.

## Solution

```bash
# confirm the rollout is stuck
kubectl -n heron rollout status deployment orders-api --timeout=15s
# error: timed out waiting for the condition

# check history
kubectl -n heron rollout history deployment orders-api
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# find the actual error — don't guess the pod name, look it up by its non-Running phase
kubectl -n heron get pods
BROKEN_POD=$(kubectl -n heron get pods -l app=orders-api --field-selector=status.phase!=Running \
  -o jsonpath='{.items[0].metadata.name}')
echo "$BROKEN_POD"
kubectl -n heron describe pod "$BROKEN_POD" | grep -iE "image:|reason|error"
# Image:  nginx:1.255-alpine
# Reason: ImagePullBackOff
# Failed to pull image "nginx:1.255-alpine": ... not found
# Error: ErrImagePull

# roll back to the last working revision
kubectl -n heron rollout undo deployment orders-api
kubectl -n heron rollout status deployment orders-api --timeout=30s
# deployment "orders-api" successfully rolled out

kubectl -n heron get deploy orders-api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# nginx:1.25-alpine
```

**What to report back:** revision 2 changed the container image to a tag that doesn't exist
(`nginx:1.255-alpine`, a typo for `1.25-alpine`), so every new Pod failed with
`ErrImagePull`/`ImagePullBackOff` and the rollout never completed — `kubectl rollout status` just
hangs until it times out. The old Pods kept serving the whole time because the default rolling
update won't take an old Pod down until a new one is available.

`kubectl rollout undo` with no `--to-revision` goes back to the immediately preceding revision,
which is what you want here. If the history had several bad revisions in a row, inspect each with
`kubectl rollout history deployment orders-api --revision=N` and pass `--to-revision=N`
explicitly.

Note that after the undo, `rollout history` shows revisions `2` and `3`, not `1` and `2`: rolling
back re-issues the old template as a new revision number rather than rewinding the counter.

## Cleanup

```bash
kubectl delete ns heron
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
