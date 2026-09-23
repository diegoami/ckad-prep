# 060 — Clone an existing stable Deployment into a canary, don't author it from scratch

**Domain:** Application Deployment · **Difficulty:** Medium

`028` writes both the `v1` and `v2` Deployments from a blank slate. Here `stable` already exists and
is running: you copy it rather than invent a new one, and the existing Service stays untouched.

## Task

> In namespace `tiber`, the storefront runs as Deployment `stable` (4 replicas, image
> `nginx:1.24-alpine`, Pod labels `app=web,version=v1`) behind Service `web-svc`, which selects
> `app=web`. Roll out a new version to about 20% of traffic: create a Deployment `canary` as a copy
> of `stable`, but with 1 replica, image `nginx:1.25-alpine` and label `version=v2`. Don't modify
> `web-svc`; it has to pick up the canary Pod on its own. Save the manifest you apply as
> `~/ckad/060/canary.yaml`.

## Documentation

What to look up: **Managing Resources** — canary deployments.
- <https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments>

## Setup

```bash
kubectl create ns tiber
mkdir -p ~/ckad/060

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stable
  namespace: tiber
spec:
  replicas: 4
  selector:
    matchLabels: {app: web, version: v1}
  template:
    metadata:
      labels: {app: web, version: v1}
    spec:
      containers:
      - name: web
        image: nginx:1.24-alpine
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: tiber
spec:
  selector: {app: web}
  ports:
  - port: 80
EOF

kubectl rollout status deployment/stable -n tiber --timeout=90s
```

## Solution

Clone the running Deployment's manifest rather than writing one by hand — this is the point of the
task, and it's the fastest path on the real exam too:
```bash
kubectl get deploy stable -n tiber -o yaml > ~/ckad/060/canary.yaml
```

Edit `canary.yaml`: change `metadata.name` to `canary`, `spec.replicas` to `1`, both
`spec.selector.matchLabels.version` and `spec.template.metadata.labels.version` to `v2`, and the
container image to `nginx:1.25-alpine`. Everything else — including the `app: web` label the
Service actually selects on — stays untouched:
```bash
sed -i \
  -e 's/name: stable/name: canary/' \
  -e 's/replicas: 4/replicas: 1/' \
  -e 's/version: v1/version: v2/g' \
  -e 's/nginx:1.24-alpine/nginx:1.25-alpine/' \
  ~/ckad/060/canary.yaml

kubectl apply -f ~/ckad/060/canary.yaml
kubectl rollout status deployment/canary -n tiber --timeout=90s
```

Confirm both Deployments are up and the Service's endpoint list picked up the canary Pod with zero
Service edits — 5 endpoints total, 4 from `stable` + 1 from `canary`:
```bash
kubectl get deploy -n tiber
# canary   1/1
# stable   4/4

kubectl get pods -n tiber --show-labels
# canary-... app=web,version=v2
# stable-... app=web,version=v1  (x4)

kubectl get endpoints web-svc -n tiber -o jsonpath='{.subsets[0].addresses[*].ip}{"\n"}'
# 5 IPs
```

**Gotcha verified live — and it's reassuring, not a trap:** `kubectl get -o yaml` dumps
`metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp`, and a full `status:` block
straight from the live object — it's tempting to think you must manually strip all of that before
re-applying under a new name, the way you would for a genuine *update*. Tested directly: applying
the cloned YAML with only `name`/`replicas`/labels/image edited, leftover `uid`/`resourceVersion`/
`status` and all, created the new Deployment without any error — the API server assigns a fresh
`uid`/`resourceVersion` on create and simply ignores the stale ones from the source object, and
`status` is a subresource that's never accepted on create in the first place. Don't waste exam time
hand-deleting those fields; only `name` (and anything that actually needs to differ) matters.

## Cleanup

```bash
kubectl delete ns tiber
rm -rf ~/ckad/060
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
