# 056 — One Ingress, multiple hosts, each to a different Service

**Domain:** Services and Networking · **Difficulty:** Medium

Unlike `055` (one host, split by path), the split here is by *virtual host*: two separate hostnames
share one Ingress, each dispatched to its own Service by the `Host` header.

## Task

> Namespace `storens` has two Deployment+Service pairs: `store` (Service port 5678) and `admin`
> (Service port 5678). Create a single Ingress `store-ingress` that routes host `store.ckad.test`
> to the `store` Service and host `admin.ckad.test` to the `admin` Service. Verify each hostname
> reaches the correct Service.

## Documentation

What to look up: **Ingress** — name-based virtual hosting.
- <https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting> —
  multiple `host` entries in one Ingress, each with its own backend.

## Setup

Requires the ingress-nginx controller on your local kind cluster, set up as in
`guide/practice-cluster.md` (check with `kubectl get pods -n ingress-nginx`).

```bash
kubectl create ns storens

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store
  namespace: storens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store
  template:
    metadata:
      labels:
        app: store
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=store-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: storens
spec:
  selector:
    app: store
  ports:
  - port: 5678
    targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
  namespace: storens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=admin-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: admin
  namespace: storens
spec:
  selector:
    app: admin
  ports:
  - port: 5678
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pod -l app=store -n storens --timeout=60s
kubectl wait --for=condition=Ready pod -l app=admin -n storens --timeout=60s
```

## Solution

Two entries in `spec.rules`, each with its own `host` and its own `backend` — no `path` trickery
needed since the split happens on the hostname, not the URL:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
  namespace: storens
spec:
  ingressClassName: nginx
  rules:
  - host: store.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: store
            port:
              number: 5678
  - host: admin.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin
            port:
              number: 5678
EOF
```

`kubectl describe ingress` confirms both host/backend pairs registered under one resource:
```bash
kubectl describe ingress store-ingress -n storens
# Rules:
#   Host             Path  Backends
#   ----             ----  --------
#   store.ckad.test
#                    /   store:5678 (...)
#   admin.ckad.test
#                    /   admin:5678 (...)
```

Verify each hostname resolves to the right Service — same host is a single IP/port, only the
`Host` header changes which backend answers:
```bash
sleep 15
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -H "Host: store.ckad.test" http://localhost/
# store-response

kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -H "Host: admin.ckad.test" http://localhost/
# admin-response
```

**Gotcha verified live:** don't expect an unrecognized `Host` header to reliably return `404`.
ingress-nginx's controller config is built from *every* Ingress object across *every* namespace on
the cluster, not scoped to `storens` the way a NetworkPolicy or RBAC Role would be — if any other
namespace has an Ingress with no `host` set (or `host: "*"`), that one becomes a catch-all and can
intercept traffic for a hostname your own Ingress never mentioned. On a shared practice cluster with one,
curling with an unrelated `Host` header returned `200` and a stock "Welcome to nginx" page from a
completely unrelated namespace's catch-all Ingress, not the `404` you'd get on a clean cluster with
only `store-ingress` present. The lesson: Ingress hostname routing is a cluster-wide namespace, so
if a task says "only these two hosts should work," verify by checking the *response body* actually
matches the Service you expect — not just that some response came back.

## Cleanup

```bash
kubectl delete ns storens
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
