# 030 — Ingress routing a host to a Service

**Domain:** Services and Networking · **Difficulty:** Medium

## Task

> A Deployment `web` and Service `web` already exist in namespace `ingns`. Create an Ingress
> `web-ingress` routing host `web.ckad.test` path `/` to that Service on port 80, and verify with
> curl.

## Documentation

What to look up: **Ingress**.
- <https://kubernetes.io/docs/concepts/services-networking/ingress/> — `rules[].host`,
  `pathType`, and the `ingressClassName` field this scenario's manifest relies on.

## Setup

```bash
# requires the ingress-nginx controller on your local kind cluster — see guide/practice-cluster.md
kubectl create ns ingns
kubectl -n ingns create deployment web --image=httpd:2.4-alpine --port=80
kubectl -n ingns expose deployment web --port=80
kubectl -n ingns rollout status deployment web --timeout=30s
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: ingns
spec:
  ingressClassName: nginx
  rules:
  - host: web.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
EOF

kubectl -n ingns get ingress web-ingress
```

**Sync delay:** curling immediately after `apply` returned `404` — the ingress
controller needs a few seconds to pick up the new Ingress object and reload its config. Waiting
~15s (`kubectl -n ingns describe ingress web-ingress` shows a `Sync` event once it's picked up)
before testing avoids a false "it's broken" read.

```bash
sleep 15
curl -H "Host: web.ckad.test" http://localhost/
# 200, httpd's default page
```

If your shell can't reach the host-mapped port 80 (some shells inside WSL or containers can't), curl
from inside the ingress controller Pod instead. It exercises the same routing:
```bash
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ingress-nginx "$INGRESS_POD" -- curl -s -o /dev/null -w "%{http_code}\n" -H "Host: web.ckad.test" http://localhost/
# 200
```

No `/etc/hosts` entry needed here — the `Host` header alone is what nginx's ingress controller
routes on; an `/etc/hosts` entry only matters if you test from a browser instead of curl.
`localhost` works because the kind cluster from `guide/practice-cluster.md` maps host ports 80/443
into the node.

## Cleanup

```bash
kubectl delete ns ingns
```
