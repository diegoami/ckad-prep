# 043 — LimitRange auto-injecting resource requests/limits on bare Pods

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

Distinct from `ResourceQuota` (`drills/drill.md` 1.9, 1.13), which caps totals across a namespace;
`LimitRange` instead fills in *per-container* defaults and enforces per-container min/max.

## Task

> Namespace `limitns` should give every container a default CPU/memory request and limit
> automatically, without every Pod author having to specify `resources:` by hand — 100m/64Mi
> request, 200m/128Mi limit. Confirm a Pod created with no `resources:` block at all actually
> inherits them.

## Documentation

What to look up: **Limit Ranges**.
- <https://kubernetes.io/docs/concepts/policy/limit-range/> — `default`/`defaultRequest` (the
  auto-fill behavior) vs. `min`/`max` (the hard gate behavior, see `082`).

## Setup

```bash
kubectl create ns limitns
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: limitns
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "64Mi"
    default:
      cpu: "200m"
      memory: "128Mi"
EOF

kubectl -n limitns run bare --image=nginx:1.25-alpine

kubectl -n limitns get pod bare -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}
```

Confirmed live: the Pod manifest sent to `kubectl run` had no `resources:` field at all, but the
object actually stored in etcd has it — `LimitRange` is a mutating admission-time default, not
something the Pod author has to remember. `default` maps to `resources.limits`, `defaultRequest`
maps to `resources.requests` — easy to get wrong, since the limits field is just called
`default`, not `defaultLimit`. A `LimitRange` also **rejects** a Pod outright if it explicitly
requests below `min` or above `max` (not shown here) — it's both a default-filler and a hard gate.

## Cleanup

```bash
kubectl delete ns limitns
```
