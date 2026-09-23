# 028 — Canary deployment via replica-ratio traffic splitting

**Domain:** Application Deployment · **Difficulty:** Hard

## Task

> Roll out version `v2` of `myapp` to roughly 20% of traffic while `v1` still serves the rest, both
> behind the same Service, without an external traffic-splitting tool — using only Kubernetes
> primitives.

## Documentation

What to look up: **Managing Resources** — the canary deployments pattern.
- <https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments> —
  the label-based approach this scenario implements, straight from the source.

## Setup

```bash
kubectl create ns canaryns
```

## Solution

Canary-by-replica-ratio works because a Service's `spec.selector` is a **label match, not a
Deployment reference** — two separate Deployments whose Pods share one label (`app: myapp`) both
land in the same Service's endpoint list. `kubectl expose` (or the Service selector) must only key
on the *shared* label, never on a Deployment-specific one — traffic then splits roughly by replica
*count*, not by any explicit percentage: 4 `v1` replicas + 1 `v2` replica ≈ 80/20.

**The mistake worth knowing about:** `kubectl create deployment` auto-sets
`spec.selector.matchLabels` to a Deployment-specific value (e.g. `app: app-v1`), and a Deployment's
`selector` is immutable — patching just `template.metadata.labels` afterward without also matching
`selector` causes `spec.template.metadata.labels: Invalid value ...: selector does not match
template`. Write both Deployments as full YAML from the start, with `selector.matchLabels` set to
match `template.metadata.labels` exactly (an `app` label the Service will select on, **plus** a
distinguishing `version` label the Deployment selector also needs but the Service ignores).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
  namespace: canaryns
spec:
  replicas: 4
  selector:
    matchLabels: {app: myapp, version: v1}
  template:
    metadata:
      labels: {app: myapp, version: v1}
    spec:
      containers:
      - name: app
        image: nginx:1.24-alpine
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
  namespace: canaryns
spec:
  replicas: 1
  selector:
    matchLabels: {app: myapp, version: v2}
  template:
    metadata:
      labels: {app: myapp, version: v2}
    spec:
      containers:
      - name: app
        image: nginx:1.25-alpine
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: canaryns
spec:
  selector: {app: myapp}     # only the shared label — no `version` here
  ports:
  - port: 80
EOF

kubectl -n canaryns rollout status deployment app-v1 --timeout=30s
kubectl -n canaryns rollout status deployment app-v2 --timeout=30s

# 5 endpoint IPs total — 4 from v1, 1 from v2, confirmed live
kubectl -n canaryns get endpoints myapp-svc -o jsonpath='{.subsets[0].addresses[*].ip}'
echo
kubectl -n canaryns get pods -l app=myapp --show-labels
```

To promote the canary to 100%, scale `app-v2` up and `app-v1` down (or to zero); to roll back,
reverse it — no Service change needed either way, since the selector never referenced `version`.

## Cleanup

```bash
kubectl delete ns canaryns
```
