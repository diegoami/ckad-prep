# 055 — One Ingress, one host, two Services split by path

**Domain:** Services and Networking · **Difficulty:** Medium

`030` routes a single host to a single Service and `049` fixes a broken single-backend Ingress.
Here one Ingress fans a single host out to two Services by path, which is what "route `/` and
`/api` correctly" questions test.

## Task

> Namespace `shopns` has two Deployment+Service pairs: `frontend` (Service port 5678) and `backend`
> (Service port 5678). Create an Ingress `shop-ingress` on host `shop.ckad.test` that routes `/` to
> `frontend` and `/api` (and anything under it, e.g. `/api/orders`) to `backend`. Verify each path
> reaches the correct Service.

## Documentation

What to look up: **Ingress** — multiple paths per host.
- <https://kubernetes.io/docs/concepts/services-networking/ingress/#simple-fanout> — the "Simple
  fanout" example is this exact shape: one host, two paths, two backend Services.

## Setup

Requires the ingress-nginx controller on your local kind cluster, set up as in
`guide/practice-cluster.md` (check with `kubectl get pods -n ingress-nginx`).

`hashicorp/http-echo` is used for both backends purely so the response body itself proves which
Service answered — no app logic to build, just `-text=<name>-response`.

```bash
kubectl create ns shopns

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: shopns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=frontend-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: shopns
spec:
  selector:
    app: frontend
  ports:
  - port: 5678
    targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: shopns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=backend-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: shopns
spec:
  selector:
    app: backend
  ports:
  - port: 5678
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pod -l app=frontend -n shopns --timeout=60s
kubectl wait --for=condition=Ready pod -l app=backend -n shopns --timeout=60s
```

## Solution

Both paths live in the same `rules[0].http.paths` list, under the one `shop.ckad.test` host —
that's the whole trick, this is one Ingress object, not two:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: shopns
spec:
  ingressClassName: nginx
  rules:
  - host: shop.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 5678
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 5678
EOF
```

`pathType: Prefix` on `/api` is what makes `/api/orders` also match `backend` — `Exact` would only
match the literal `/api` and let `/api/orders` fall through to the next matching rule (here, `/`,
so it would wrongly land on `frontend`).

Verify each path resolves to the right Service. Curl from inside the ingress controller pod (same
pattern as `030`/`049`) rather than relying on the host-mapped `localhost:80`, which isn't
reachable from every shell:

```bash
sleep 15
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -H "Host: shop.ckad.test" http://localhost/
# frontend-response

kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -H "Host: shop.ckad.test" http://localhost/api
# backend-response

kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -H "Host: shop.ckad.test" http://localhost/api/orders
# backend-response — Prefix match covers the sub-path too
```

**Gotcha verified live:** the order of the two path entries in the YAML doesn't matter. Rules
written `/` before `/api` and rules written `/api` before `/` were both tested and produced
identical routing — ingress-nginx matches by longest-prefix, not declaration order, so you don't
need to worry about listing the more specific path first the way you might with e.g. a firewall
rule list.

## Cleanup

```bash
kubectl delete ns shopns
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
