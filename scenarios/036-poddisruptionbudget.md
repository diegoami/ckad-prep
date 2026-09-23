# 036 — Protect a Deployment from voluntary disruption with a PodDisruptionBudget

**Domain:** Application Deployment · **Difficulty:** Easy

## Task

> Deployment `web` in namespace `pdbns` runs 3 replicas. A node drain or voluntary eviction should
> never be allowed to take it below 2 available replicas at once. Enforce that.

## Documentation

What to look up: **Specifying a Disruption Budget for your Application**.
- <https://kubernetes.io/docs/tasks/run-application/configure-pdb/> — `minAvailable` vs.
  `maxUnavailable`, and that a PDB only governs *voluntary* disruptions (drains/evictions), not node failures.

## Setup

```bash
kubectl create ns pdbns
kubectl -n pdbns create deployment web --image=nginx:1.25-alpine --replicas=3
kubectl -n pdbns rollout status deployment web --timeout=30s
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: pdbns
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: web}
EOF

kubectl -n pdbns get pdb web-pdb
# ALLOWED DISRUPTIONS shows 1 — with 3 replicas and minAvailable 2, exactly one can be
# voluntarily evicted at a time
```

A `PodDisruptionBudget` only governs **voluntary** disruption (`kubectl drain`, a cluster-autoscaler
scale-down, eviction API calls) — it does nothing for involuntary disruption (a node crashing, a
Pod OOMKilled). `minAvailable: 2` and `maxUnavailable: 1` are two ways to express the same 3-replica
constraint here; use whichever the question phrases it as, they're not always interchangeable at
other replica counts (`maxUnavailable` scales with replica count changes, `minAvailable` doesn't).

Confirmed live: `kubectl -n pdbns get pdb web-pdb` immediately after creation shows
`ALLOWED DISRUPTIONS: 1`, matching `3 replicas - minAvailable 2`.

## Cleanup

```bash
kubectl delete ns pdbns
```
