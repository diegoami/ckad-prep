# 049 — Fix an Ingress pointing at the wrong Service, without recreating it

**Domain:** Services and Networking · **Difficulty:** Medium

Unlike 030, which creates an Ingress from scratch, this is troubleshooting: the Ingress already
exists, but a typo in its backend makes it answer `503`.

## Task

> The product catalogue in namespace `rhine` runs as Deployment `catalog` behind Service `catalog`
> (port 8080). An Ingress `catalog-ingress` is supposed to publish it on host `catalog.ckad.test`,
> but every request comes back `503 Service Temporarily Unavailable`. Fix the Ingress so the host
> serves the app. Leave the Deployment and the Service as they are.

## Documentation

What to look up: **Ingress** — the backend service/port fields.
- <https://kubernetes.io/docs/concepts/services-networking/ingress/> — `backend.service.name`/
  `backend.service.port.number`, and how `kubectl describe ingress` surfaces backend resolution errors.

## Setup

Requires the ingress-nginx controller on your local kind cluster, set up as in
`guide/practice-cluster.md` (check with `kubectl get pods -n ingress-nginx`).

```bash
kubectl create ns rhine
kubectl create deployment catalog --image=nginx --port=80 -n rhine
kubectl expose deployment catalog -n rhine --port=8080 --target-port=80
kubectl wait --for=condition=Ready pod -l app=catalog -n rhine --timeout=60s

# Deliberately broken: backend Service name has a typo ("catalog-svc" doesn't exist)
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog-ingress
  namespace: rhine
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: catalog.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: catalog-svc
            port:
              number: 8080
EOF
```

Confirm the "given" broken state (give the controller ~15s to pick up the new Ingress first):
```bash
sleep 15
kubectl describe ingress catalog-ingress -n rhine
# Backends: catalog-svc:8080 (<error: services "catalog-svc" not found>)

INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -o /dev/null -w "%{http_code}\n" -H "Host: catalog.ckad.test" http://localhost/
# 503
```

## Solution

`kubectl describe ingress` already names the exact problem — the backend's Service name doesn't
exist. Cross-check against what's actually there:
```bash
kubectl get svc -n rhine
# NAME      ...
# catalog   ...    <- the real name, not "catalog-svc"
```

Patch just the broken field, don't recreate the Ingress:
```bash
kubectl patch ingress catalog-ingress -n rhine --type=json \
  -p='[{"op": "replace", "path": "/spec/rules/0/http/paths/0/backend/service/name", "value": "catalog"}]'
```
**Faster by hand:** `kubectl edit ingress catalog-ingress -n rhine`, fix `service.name` under
`backend` directly, save — same result without the JSON-patch array-index path.

Confirm:
```bash
sleep 5
kubectl describe ingress catalog-ingress -n rhine
# Backends: catalog:8080 (10.244.x.x:80)

INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -o /dev/null -w "%{http_code}\n" -H "Host: catalog.ckad.test" http://localhost/
# 200
```

**Gotcha verified live:** a *wrong port number* on the backend (e.g. `80` instead of the Service's
actual `8080`) does **not** reliably reproduce this failure the way a wrong Service name does —
ingress-nginx falls back to the Service's only port when there's just one defined, so a
single-port Service silently tolerates a mismatched port number in the Ingress. If you're
authoring your own broken-Ingress practice and want a guaranteed failure, break the Service *name*
(unresolvable) or `pathType` (e.g. `Exact` on a path clients hit with extra segments), not just the
port number on a single-port Service.

## Cleanup

```bash
kubectl delete ns rhine
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
