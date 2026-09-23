# 077 — 50%/50% rolling-update strategy, a bad image change, rollout status/history, rollback

**Domain:** Application Deployment · **Difficulty:** Medium

`044` tunes the strategy with a healthy image throughout, and `066` is a rollback with no strategy
tuning. This one chains both, and the image change is deliberately one that can never become
ready, so `rollout status`, `rollout history` and `rollout undo` all have something to show.

## Task

> In namespace `crete`, the `storefront` Deployment runs 4 replicas of `nginx:1.24-alpine`.
> Configure its rolling update so that up to half the desired Pods can be added and up to half
> can be unavailable during a rollout (`maxSurge: 50%`, `maxUnavailable: 50%`). Then switch its
> container image to `alpine`, check how the rollout is going and what its revision history
> looks like, and finally return the Deployment to the revision it was on before the image
> change.

## Documentation

What to look up: **Deployments** — rolling update strategy and rollback, together.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment>
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment>

## Setup

```bash
kubectl create ns crete
kubectl create deployment storefront -n crete --image=nginx:1.24-alpine --replicas=4
kubectl rollout status deployment/storefront -n crete --timeout=60s
```

## Solution

Set the strategy first — both fields live under `spec.strategy.rollingUpdate`:
```bash
kubectl patch deployment storefront -n crete --type=json \
  -p='[{"op":"replace","path":"/spec/strategy/rollingUpdate/maxSurge","value":"50%"},
       {"op":"replace","path":"/spec/strategy/rollingUpdate/maxUnavailable","value":"50%"}]'
```
**Faster by hand:** `kubectl edit deployment storefront -n crete`, change both `maxSurge:` and
`maxUnavailable:` under `spec.strategy.rollingUpdate` in one pass, save — quicker than a two-op JSON patch.

Change the image — `kubectl create deployment --image=nginx:1.24-alpine` names the container after
the image's *repository*, not the full reference, so it's `nginx`, not `nginx:1.24-alpine` or
`storefront`:
```bash
kubectl get deploy storefront -n crete -o jsonpath='{.spec.template.spec.containers[0].name}{"\n"}'
# nginx

kubectl set image deployment/storefront nginx=alpine -n crete
```

**Watch the rollout status — this is the part the task is actually testing:**
```bash
kubectl rollout status deployment/storefront -n crete --timeout=15s
# error: timed out waiting for the condition

kubectl get pods -n crete
# several new-image Pods cycling between Completed and CrashLoopBackOff, 2 old Pods still Running
```
`maxSurge: 50%` + `maxUnavailable: 50%` on 4 replicas means up to 4 Pods can be mid-replacement
simultaneously (2 surge + 2 unavailable) — all 4 new-revision Pods get created at once rather than
one at a time. None of them stabilize: bare `alpine`'s default entrypoint is a shell with no
foreground process, so every replacement Pod exits immediately and gets restarted, `READY` never
reaches `4/4`, and `rollout status` just times out waiting.

Check history — the failed rollout still produced a new revision, it just never finished:
```bash
kubectl rollout history deployment/storefront -n crete
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

Roll back:
```bash
kubectl rollout undo deployment/storefront -n crete
kubectl rollout status deployment/storefront -n crete --timeout=60s
# deployment "storefront" successfully rolled out

kubectl get pods -n crete
# 4/4 Running, back on nginx:1.24-alpine
```

**Lesson:** the "change the image to alpine" step isn't a slip — bare `alpine` (or any base OS
image with no long-running foreground process) is a deliberately bad image change, and the point of
walking through `rollout status` → `rollout history` → `rollout undo` is to *observe* a rollout that
can't succeed and recover from it, not to end up with `alpine` actually running. If a task asked
for `alpine` to end up genuinely serving traffic, it would need an explicit `command`/`args` keeping
a process alive (same fix as `073`) — that's a different, separate ask from "see the rollout
struggle and roll it back."

## Cleanup

```bash
kubectl delete ns crete
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
