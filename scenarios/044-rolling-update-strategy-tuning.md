# 044 — Tune rolling-update strategy so a Deployment never dips below full capacity

**Domain:** Application Deployment · **Difficulty:** Medium

Scenario 001 rolls back a Deployment; this one controls *how* a rollout itself proceeds.

## Task

> Deployment `web` (namespace `rollns`, 4 replicas) must never have fewer than 4 Pods `Ready`
> during a rolling update — capacity can temporarily go *above* 4 while the new version comes up,
> but never below. Configure that, then trigger an image update and confirm the behavior.

## Documentation

What to look up: **Deployments** — the rolling update strategy fields.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment> —
  `maxSurge`/`maxUnavailable` semantics and defaults (25%/25%).

## Setup

```bash
kubectl create ns rollns
kubectl -n rollns create deployment web --image=nginx:1.24-alpine --replicas=4
kubectl -n rollns rollout status deployment web --timeout=60s
```

## Solution

```bash
kubectl -n rollns get deployment web -o jsonpath='{.spec.strategy}{"\n"}'
# {"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"}

kubectl -n rollns patch deployment web \
  -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
# or: kubectl -n rollns edit deployment web  → spec.strategy.rollingUpdate

kubectl -n rollns set image deployment/web nginx=nginx:1.25-alpine
kubectl -n rollns get rs -w     # watch in a second pane; Ctrl-C when done
```

`maxUnavailable: 0` is the field that actually enforces the "never below 4" requirement;
`maxSurge` only controls how much *extra* capacity is allowed while the swap happens (bigger surge
= faster rollout, more resource usage at peak). Confirmed live via `kubectl -n rollns get rs -w`
during the rollout: the new ReplicaSet scales up to 1 *before* the old one scales down at all, and
the old ReplicaSet's count never drops until a replacement is already `Ready` — total capacity
stays at 4 or 5 throughout, never 3.

```bash
kubectl -n rollns rollout status deployment web --timeout=30s
# deployment "web" successfully rolled out
```

The defaults (`maxSurge: 25%`, `maxUnavailable: 25%`) allow a brief capacity dip during rollout —
fine for most apps, but the wrong choice for anything that can't tolerate reduced capacity even
momentarily (rate-limited backends, low-replica-count services where 25% is a whole Pod).

## Cleanup

```bash
kubectl delete ns rollns
```
