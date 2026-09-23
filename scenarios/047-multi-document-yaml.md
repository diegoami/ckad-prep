# 047 — Author a ConfigMap, Deployment, and Service as one multi-document YAML file

**Domain:** Application Deployment · **Difficulty:** Easy

The other scenarios apply each resource as its own `kubectl apply -f -` heredoc; this one is
about the `---`-separated multi-document style itself, and getting cross-references between the
documents right in one shot.

## Task

> Write a single file `app.yaml` containing a ConfigMap `app-cfg` (key `greeting=hello`), a
> Deployment `app` that loads it via `envFrom`, and a Service `app-svc` exposing it on port 80 — in
> namespace `multidocns` — and apply all three with one `kubectl apply -f` call.

## Documentation

What to look up: **Managing Kubernetes Objects** — organizing resource config.
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/> — why
  `---`-separated multi-document YAML in one file is a normal, supported `kubectl apply -f` input.

## Setup

```bash
kubectl create ns multidocns
```

## Solution

```bash
cat > app.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-cfg
  namespace: multidocns
data:
  greeting: hello
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: multidocns
spec:
  replicas: 1
  selector: {matchLabels: {app: app}}
  template:
    metadata: {labels: {app: app}}
    spec:
      containers:
      - name: app
        image: nginx:1.25-alpine
        envFrom:
        - configMapRef: {name: app-cfg}
---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: multidocns
spec:
  selector: {app: app}
  ports:
  - port: 80
EOF

kubectl apply -f app.yaml
kubectl -n multidocns get cm,deploy,svc
```

`---` on its own line is the YAML document separator — `kubectl apply -f` (and `create -f`) walks
every document in the file and applies each as its own object, in the order they appear. Order
matters for legibility but not for the apply itself: the API server doesn't care that the
ConfigMap is declared before the Deployment that references it, since they're all sent as separate
API calls, not resolved as one dependency graph — but a Pod that starts *before* its
`envFrom`/`configMapRef` target exists would still hit the same "missing ConfigMap" stall as
scenario 015, so declaring dependencies first is good practice even though it isn't enforced.

## Cleanup

```bash
kubectl delete ns multidocns
rm -f app.yaml
```
