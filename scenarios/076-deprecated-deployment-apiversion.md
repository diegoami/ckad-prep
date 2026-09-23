# 076 — Convert a Deployment off a removed apiVersion, and the selector it never had

**Domain:** Application Observability and Maintenance · **Difficulty:** Medium

Unlike `031` (an Ingress) and `067` (an HPA), a Deployment on `extensions/v1beta1` has a structural
gap: that old API let `spec.selector` be omitted and derived it from the Pod template's labels,
which `apps/v1` no longer does. So bumping only the `apiVersion` line still doesn't apply, for a
different reason than in those two scenarios.

## Task

> The invoicing team in namespace `bali` dug up the manifest for their `invoice-api` Deployment
> from an old repository and saved it as `~/ckad/076/invoice-api.yaml`. It was written for
> `extensions/v1beta1`, which this cluster's API server no longer serves. Update the manifest
> so it uses the current stable API for Deployments, apply it, and make sure both replicas are
> running.
>
> ```yaml
> apiVersion: extensions/v1beta1
> kind: Deployment
> metadata:
>   name: invoice-api
>   namespace: bali
> spec:
>   replicas: 2
>   template:
>     metadata:
>       labels:
>         app: invoice-api
>     spec:
>       containers:
>       - name: api
>         image: nginx:1.24-alpine
> ```

## Documentation

What to look up: **Deprecated API Migration Guide**.
- <https://kubernetes.io/docs/reference/using-api/deprecation-guide/> — `extensions/v1beta1` and
  `apps/v1beta1`/`v1beta2` Deployment removal, and the current `apps/v1` shape (including the
  now-mandatory `spec.selector`).

## Setup

```bash
kubectl create ns bali
mkdir -p ~/ckad/076

cat > ~/ckad/076/invoice-api.yaml <<'EOF'
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: invoice-api
  namespace: bali
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: invoice-api
    spec:
      containers:
      - name: api
        image: nginx:1.24-alpine
EOF
```

## Solution

Confirm it's rejected — `extensions/v1beta1` Deployments were removed in Kubernetes 1.16, long
before this cluster's version:
```bash
cd ~/ckad/076
kubectl apply -f invoice-api.yaml
# error: resource mapping not found for name: "invoice-api" ... no matches for kind "Deployment"
# in version "extensions/v1beta1"
```

**Bumping only the apiVersion line is not enough** — this manifest never had a `spec.selector` at
all, which `extensions/v1beta1` tolerated by silently deriving one from the Pod template's labels.
`apps/v1` requires it explicitly:
```bash
sed 's/extensions\/v1beta1/apps\/v1/' invoice-api.yaml > half-fixed.yaml
kubectl apply -f half-fixed.yaml
```
```
The Deployment "invoice-api" is invalid:
* spec.selector: Required value
* spec.template.metadata.labels: Invalid value: map[string]string{"app":"invoice-api"}: `selector` does not match template `labels`
```
(Kubernetes v1.30 prints the labels as `map[string]string{...}`; newer versions print
`{"app":"invoice-api"}`. Same error either way.)

Both errors point at the same missing field — add `spec.selector.matchLabels`, and it must match
`spec.template.metadata.labels` exactly (same requirement every `apps/v1` Deployment has, just
invisible on the old API where it was implicit):
```bash
cat > invoice-api.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: invoice-api
  namespace: bali
spec:
  replicas: 2
  selector:
    matchLabels:
      app: invoice-api
  template:
    metadata:
      labels:
        app: invoice-api
    spec:
      containers:
      - name: api
        image: nginx:1.24-alpine
EOF

kubectl apply -f invoice-api.yaml
kubectl rollout status deployment/invoice-api -n bali --timeout=60s
# deployment "invoice-api" successfully rolled out
cd ~
```

The two structural changes `extensions/v1beta1` → `apps/v1` always needs on a Deployment: the
version line itself, and an explicit `spec.selector` that matches the Pod template's labels — the
second one is easy to miss because the old manifest never had to have it, so there's no field to
"notice is wrong," only a field that's silently absent.

## Cleanup

```bash
kubectl delete ns bali
rm -rf ~/ckad/076
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
