# 058 — Edit an existing Deployment's mismatched limits, then scale under a ResourceQuota

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Hard

In `052` the container has *no* limits and the fix is adding them. Here it already has requests
**and** limits, just at the wrong ratio: you edit existing values in place and then scale, and the
order you do it in matters.

## Task

> In namespace `vistula`, the Deployment `api` runs 1 replica and its container sets both
> `resources.requests` and `resources.limits`. The namespace's ResourceQuota `vistula-quota` must not
> be changed. Scale `api` to 3 replicas and make sure all 3 are running. Company policy says the
> container's limits must be exactly double its requests.

## Documentation

What to look up: **Resource Quotas**, plus **Deployments**' rolling-update surge behavior.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment> —
  why a rolling update needs *surge* headroom on top of what's already running.

## Setup

```bash
kubectl create ns vistula

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: vistula-quota
  namespace: vistula
spec:
  hard:
    requests.cpu: 600m
    requests.memory: 768Mi
    limits.cpu: 1200m
    limits.memory: 1536Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: vistula
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          # Deliberately broken: limits are 3x requests, not 2x — fine at 1 replica,
          # but won't fit the quota once scaled to 3
          limits:
            cpu: 600m
            memory: 768Mi
EOF

kubectl rollout status deployment/api -n vistula --timeout=60s
```

Confirm the "given" state — healthy at 1 replica, quota has plenty of headroom for now:
```bash
kubectl get deploy api -n vistula
# READY 1/1
```

## Solution

Scaling first, before fixing anything, shows exactly why the ratio matters — don't skip this step,
it's what makes the quota math concrete instead of abstract:
```bash
kubectl scale deployment api -n vistula --replicas=3
sleep 5
kubectl get deploy api -n vistula
# READY 2/3 — stuck

RS=$(kubectl get rs -n vistula -l app=api -o jsonpath='{.items[0].metadata.name}')
kubectl describe rs "$RS" -n vistula | grep FailedCreate
# forbidden: exceeded quota: vistula-quota, requested: limits.cpu=600m,limits.memory=768Mi,
#   used: limits.cpu=1200m,limits.memory=1536Mi, limited: limits.cpu=1200m,limits.memory=1536Mi
```
2 replicas at 600m/768Mi limits each already consume the *entire* `limits.cpu`/`limits.memory`
quota (1200m/1536Mi, exactly the hard cap), so there's no room for a 3rd Pod.

**Scale back down before patching, then fix, then scale up again** — patching the container spec
while replicas are stuck creates a new ReplicaSet, and a rolling update needs *surge* headroom on
top of what the old (already quota-maxed) Pods hold, which deadlocks it instead of fixing anything:
```bash
kubectl scale deployment api -n vistula --replicas=1
sleep 3

kubectl patch deployment api -n vistula --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits", "value": {"cpu": "400m", "memory": "512Mi"}}]'
kubectl rollout status deployment/api -n vistula --timeout=60s
```
**Faster by hand:** `kubectl edit deployment api -n vistula`, overwrite the `limits:` values
under `resources:` directly, save.

Then scale up again:
```bash
kubectl scale deployment api -n vistula --replicas=3
kubectl rollout status deployment/api -n vistula --timeout=60s
# deployment "api" successfully rolled out
```

Confirm — 3 replicas at the corrected 400m/512Mi limits (double the 200m/256Mi requests) land
*exactly* on the quota's hard ceiling:
```bash
kubectl get deploy api -n vistula
# READY 3/3

kubectl describe resourcequota vistula-quota -n vistula | grep -A5 "Resource "
# limits.cpu       1200m   1200m
# limits.memory    1536Mi  1536Mi
# requests.cpu     600m    600m
# requests.memory  768Mi   768Mi
```

**Gotcha verified live:** patching the resources *before* scaling back down (i.e. patching while
still stuck at 2/3) triggers a rolling update that can't proceed at all — the old ReplicaSet's Pods
already hold 100% of the `limits.*` quota, so the new ReplicaSet can't even create one surge Pod,
and `kubectl rollout status` just times out. Always shrink back to a replica count the *current*
(broken) spec can afford before editing the template, then grow again once the template is fixed —
editing-in-place under quota pressure is not the same problem as authoring correctly from scratch.

## Cleanup

```bash
kubectl delete ns vistula
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
