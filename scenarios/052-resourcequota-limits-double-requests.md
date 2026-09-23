# 052 — Fix a Deployment rejected by a ResourceQuota, by setting limits to double the requests

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

Unlike `043-limitrange-defaults.md`, where a LimitRange injects defaults, nothing fills in the
missing values here: the quota stays as it is and the fix is entirely on the workload.

## Task

> In namespace `loire`, the Deployment `worker` never gets any Pods: it sits at 0 available
> replicas. The namespace has a ResourceQuota `loire-quota` covering CPU and memory requests and
> limits, and you may not change it. Fix `worker` so its Pod starts, giving the container limits of
> exactly twice its current requests.

## Documentation

What to look up: **Resource Quotas**.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/> — the "Requests compared to
  Limits" note explaining why setting `limits.*` in a quota makes every container's limit mandatory.

## Setup

```bash
kubectl create ns loire

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: loire-quota
  namespace: loire
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF

# Deliberately broken: requests set, but no limits at all
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: loire
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: nginx
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
EOF
```

Confirm the "given" broken state:
```bash
sleep 3
kubectl get deploy worker -n loire
# READY 0/1

RS=$(kubectl get rs -n loire -l app=worker -o jsonpath='{.items[0].metadata.name}')
kubectl describe rs "$RS" -n loire | grep FailedCreate
# Error creating: pods "worker-..." is forbidden: failed quota: loire-quota:
#   must specify limits.cpu for: worker; limits.memory for: worker
```
Why the ReplicaSet and not the Deployment: a Deployment never talks to the Pods API directly — it
manages a ReplicaSet, and it's the *ReplicaSet controller* that actually calls "create this Pod" and
gets the quota rejection back. That rejection is recorded as an Event on the object that made the
failing API call, which is the ReplicaSet, not the Deployment. `kubectl describe deployment worker`
only shows a rollup (e.g. a Progressing/ReplicaFailure condition) — the specific `FailedCreate`
message with the actual quota numbers lives on the ReplicaSet.

## Solution

The quota has `limits.cpu`/`limits.memory` set, which means **every Pod must declare a limit for
every container**, even if the quota's own limit ceiling wouldn't otherwise be hit — a quota with
`limits.*` present makes limits mandatory, not just capped. Patch the Deployment's container to add
limits at exactly double the existing requests (250m/256Mi → 500m/512Mi):
```bash
kubectl patch deployment worker -n loire --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/resources/limits", "value": {"cpu": "500m", "memory": "512Mi"}}]'
```
**Faster by hand:** `kubectl edit deployment worker -n loire`, add a `limits:` block right under
the existing `requests:` in the container's `resources:`, save.

Confirm:
```bash
kubectl rollout status deployment/worker -n loire --timeout=60s
kubectl get pods -n loire
# worker-... 1/1 Running
```

The requests (250m/256Mi) stay within the quota's `requests.*` ceiling (1 CPU/1Gi), and the new
limits (500m/512Mi) stay within `limits.*` (2 CPU/2Gi) — "double the requests" is a safe rule of
thumb for satisfying a quota like this without knowing the workload's real ceiling, not a
Kubernetes-enforced rule.

## Cleanup

```bash
kubectl delete ns loire
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
