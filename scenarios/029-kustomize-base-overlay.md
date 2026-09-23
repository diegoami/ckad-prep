# 029 — Kustomize base + dev overlay with a replica-count patch

**Domain:** Application Deployment · **Difficulty:** Medium

`examples/kustomize/` has more elaborate Kustomize examples to browse; this is the minimal
version of the same idea.

## Task

> Write a Kustomize `base` with a single-replica nginx Deployment named `app`, and a `dev` overlay
> that prefixes resource names with `dev-` and bumps replicas to 2 — without editing the base
> itself. Apply the overlay into namespace `kustns`.

## Documentation

What to look up: **Declarative Management of Kubernetes Objects Using Kustomize**.
- <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/> — `bases`,
  `overlays`, and the patch/override mechanics `kubectl apply -k` uses.

## Setup

```bash
mkdir -p ~/ckad/029/base ~/ckad/029/overlays/dev
cd ~/ckad/029
kubectl create ns kustns
```

## Solution

```bash
cat > base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels: {app: app}
  template:
    metadata:
      labels: {app: app}
    spec:
      containers:
      - name: app
        image: nginx:1.25-alpine
EOF

cat > base/kustomization.yaml <<'EOF'
resources:
- deployment.yaml
EOF

cat > overlays/dev/kustomization.yaml <<'EOF'
namePrefix: dev-
resources:
- ../../base
patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 2
EOF

# preview the merged output before applying
kubectl kustomize overlays/dev

kubectl apply -k overlays/dev -n kustns
kubectl -n kustns get deploy
# dev-app, 2 replicas — base/deployment.yaml itself was never touched
```

`patches:` is the current field and accepts both JSON 6902 operations (as here, with a `target:`)
and strategic-merge snippets. Older Kustomize examples you'll find online use the separate
`patchesStrategicMerge:` / `patchesJson6902:` fields instead — those are deprecated, and
`kustomize edit fix` rewrites them to `patches:`.

## Cleanup

```bash
kubectl delete ns kustns
rm -rf ~/ckad/029
```
