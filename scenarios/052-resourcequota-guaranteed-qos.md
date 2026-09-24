# 052 — A quota that insists on limits: get a Deployment's Pods created, with Guaranteed QoS

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

Unlike `043-limitrange-defaults.md`, where a LimitRange injects defaults, nothing fills in the
missing values here: the quota stays as it is and the fix is entirely on the workload.

## Task

> The `mahleb` team rolled out Deployment `thumbnailer` this morning, and it still hasn't produced
> a single Pod. The namespace carries a ResourceQuota, `mahleb-compute`, which you must leave
> alone. Change `thumbnailer` so its Pod is created and running, and so that the Pod lands in the
> `Guaranteed` QoS class. Keep the container's current resource requests as they are.

## Documentation

What to look up: **Resource Quotas**, plus **Pod Quality of Service Classes**.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/> — the "Requests compared to
  Limits" note explaining why setting `limits.*` in a quota makes every container's limit mandatory.
- <https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/> — the exact rule for `Guaranteed`.

## Setup

```bash
kubectl create ns mahleb

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mahleb-compute
  namespace: mahleb
spec:
  hard:
    pods: "4"
    requests.cpu: 800m
    requests.memory: 1Gi
    limits.cpu: 800m
    limits.memory: 1Gi
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thumbnailer
  namespace: mahleb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thumbnailer
  template:
    metadata:
      labels:
        app: thumbnailer
    spec:
      containers:
      - name: thumbnailer
        image: httpd:2.4-alpine
        resources:
          requests:
            cpu: 150m
            memory: 192Mi
EOF
```

Confirm the "given" broken state:
```bash
sleep 3
kubectl get deploy thumbnailer -n mahleb
# READY 0/1

RS=$(kubectl get rs -n mahleb -l app=thumbnailer -o jsonpath='{.items[0].metadata.name}')
kubectl describe rs "$RS" -n mahleb | grep FailedCreate
# Error creating: pods "thumbnailer-..." is forbidden: failed quota: mahleb-compute:
#   must specify limits.cpu for: thumbnailer; limits.memory for: thumbnailer
```
Why the ReplicaSet and not the Deployment: a Deployment never talks to the Pods API directly. It
manages a ReplicaSet, and it's the *ReplicaSet controller* that calls "create this Pod" and gets the
quota rejection back. The rejection is recorded as an Event on the object that made the failing API
call, which is the ReplicaSet. `kubectl describe deployment thumbnailer` only shows a rollup (a
`ReplicaFailure` condition); the `FailedCreate` message with the quota details lives on the
ReplicaSet.

## Solution

The quota lists `limits.cpu` and `limits.memory`, which means **every new Pod must declare a limit
for both, on every container**. A quota that tracks `limits.*` makes limits mandatory, not just
capped, even when the Pod would fit easily.

For `Guaranteed`, every container needs CPU and memory limits, and each request has to equal its
limit. The requests stay at 150m/192Mi, so the limits are exactly those values. `kubectl set
resources` is the dedicated command for this:
```bash
kubectl set resources deployment thumbnailer -n mahleb -c thumbnailer \
  --limits=cpu=150m,memory=192Mi
```
The same change as a patch, for a script:
```bash
kubectl patch deployment thumbnailer -n mahleb --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/resources/limits", "value": {"cpu": "150m", "memory": "192Mi"}}]'
```
Running both is harmless (the second changes nothing). **Faster by hand:** `kubectl edit deployment
thumbnailer -n mahleb`, add a `limits:` block next to the existing `requests:`, save.

Confirm:
```bash
kubectl rollout status deployment/thumbnailer -n mahleb --timeout=60s
kubectl get pods -n mahleb -l app=thumbnailer \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,QOS:.status.qosClass
# thumbnailer-...   Running   Guaranteed

kubectl describe resourcequota mahleb-compute -n mahleb
# limits.cpu       150m   800m
# limits.memory    192Mi  1Gi
# pods             1      4
# requests.cpu     150m   800m
# requests.memory  192Mi  1Gi
```

Two ways to get this subtly wrong:
- Limits above the requests (say 300m/384Mi) also satisfy the quota and the Pod runs, but its QoS
  class is `Burstable`, not `Guaranteed`. Check `.status.qosClass`, don't assume it.
- Setting only `limits` and deleting the `requests` block would also give `Guaranteed`, because a
  missing request defaults to the limit. The task says to keep the requests, so leave them in place.

## Cleanup

```bash
kubectl delete ns mahleb
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
