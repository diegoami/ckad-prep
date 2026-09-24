# 058 — Over-sized limits block a scale-out under a quota: shrink, fix, then grow

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Hard

In `052` the container has *no* limits and the fix is adding them. Here it already has requests
**and** limits, the limits are just far too generous. You change existing values in place and
scale, and the order you do it in matters.

## Task

> Deployment `ledger-sync` in namespace `galangal` runs a single replica today. Ahead of month-end
> it has to run 4 replicas, all of them Running. The namespace's ResourceQuota `galangal-budget` is
> fixed and must not be edited. A platform rule, which the current Deployment breaks, says a
> container's CPU and memory limits may be at most 1.5 times its requests. Leave the requests
> unchanged.

## Documentation

What to look up: **Resource Quotas**, plus **Deployments**' rolling-update surge behavior.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment> —
  why a rolling update needs *surge* headroom on top of what's already running.

## Setup

```bash
kubectl create ns galangal

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: galangal-budget
  namespace: galangal
spec:
  hard:
    requests.cpu: 500m
    requests.memory: 640Mi
    limits.cpu: 600m
    limits.memory: 768Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ledger-sync
  namespace: galangal
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ledger-sync
  template:
    metadata:
      labels:
        app: ledger-sync
    spec:
      containers:
      - name: ledger-sync
        image: httpd:2.4-alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 400m
            memory: 512Mi
EOF

kubectl rollout status deployment/ledger-sync -n galangal --timeout=60s
```

Confirm the "given" state: healthy at 1 replica.
```bash
kubectl get deploy ledger-sync -n galangal
# READY 1/1
```

## Solution

Work out the numbers first. The rule caps the limits at 1.5 × 100m = **150m** CPU and
1.5 × 128Mi = **192Mi** memory. At 4 replicas that is 600m/768Mi of limits, exactly the quota's
`limits.*` ceiling, and 400m/512Mi of requests, inside `requests.*`. The current 400m/512Mi limits
cost 4× the requests, so one replica already uses two thirds of the `limits.cpu` budget.

Scaling first shows the problem concretely:
```bash
kubectl scale deployment ledger-sync -n galangal --replicas=4
sleep 5
kubectl get deploy ledger-sync -n galangal
# READY 1/4 — stuck

RS=$(kubectl get rs -n galangal -l app=ledger-sync -o jsonpath='{.items[0].metadata.name}')
kubectl describe rs "$RS" -n galangal | grep FailedCreate | tail -1
# ... forbidden: exceeded quota: galangal-budget, requested: limits.cpu=400m,limits.memory=512Mi,
#   used: limits.cpu=400m,limits.memory=512Mi, limited: limits.cpu=600m,limits.memory=768Mi
```
One Pod at 400m leaves 200m, and a second one needs 400m, so nothing more gets created.

**Scale back down before changing the template, then fix, then scale up again.** Changing the Pod
template starts a rolling update, which needs quota headroom for new Pods *on top of* the old ones.
At one replica there is room for a single 150m surge Pod (400m + 150m = 550m):
```bash
kubectl scale deployment ledger-sync -n galangal --replicas=1
sleep 3

kubectl set resources deployment ledger-sync -n galangal -c ledger-sync \
  --limits=cpu=150m,memory=192Mi
kubectl rollout status deployment/ledger-sync -n galangal --timeout=60s
```
`kubectl set resources` only touches the values you name, so the requests stay as they are. The
JSON-patch equivalent, if you prefer one:
`kubectl patch deployment ledger-sync -n galangal --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits","value":{"cpu":"150m","memory":"192Mi"}}]'`.
**Faster by hand:** `kubectl edit deployment ledger-sync -n galangal`, overwrite the two values
under `limits:`, save.

Then scale out:
```bash
kubectl scale deployment ledger-sync -n galangal --replicas=4
kubectl rollout status deployment/ledger-sync -n galangal --timeout=60s
# deployment "ledger-sync" successfully rolled out
```

Confirm: 4 replicas at 150m/192Mi land exactly on the `limits.*` ceiling.
```bash
kubectl get deploy ledger-sync -n galangal
# READY 4/4

kubectl describe resourcequota galangal-budget -n galangal | grep -A5 "Resource "
# limits.cpu       600m   600m
# limits.memory    768Mi  768Mi
# requests.cpu     400m   500m
# requests.memory  512Mi  640Mi
```

**Gotcha verified live:** changing the limits while the Deployment is still stuck at 1/4 doesn't
recover it. The rollout does create a new Pod, but then old and new Pods share the quota: the old
400m Pod stays up because taking it away would leave too few available replicas, and there isn't
enough budget for the rest of the new ones. `kubectl rollout status` times out and the Deployment
sits at 2/4 until you scale it back down. Shrink to a replica count the *current* spec can afford
before editing the template, and grow once the template is fixed.

## Cleanup

```bash
kubectl delete ns galangal
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
