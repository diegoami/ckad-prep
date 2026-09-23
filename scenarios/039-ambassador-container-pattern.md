# 039 — Ambassador container pattern: proxy localhost traffic to a real Service

**Domain:** Application Design and Build · **Difficulty:** Medium

The curriculum names multi-container patterns as "sidecar, init and others"; scenarios 016 and
017 cover sidecar and init, this covers the ambassador pattern.

## Task

> A Deployment `backend` + Service already exist in namespace `ambassador`. Add a Pod
> `app-with-ambassador` where the main container only ever talks to `localhost:8080` — it should
> have no idea the real backend is a Service named `backend` on a different port. An ambassador
> container in the same Pod does the actual proxying.

## Documentation

What to look up: **Pods** — how containers in one Pod share network/localhost.
- <https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers> — the
  shared network namespace fact the ambassador pattern (proxy on localhost) depends on; the
  ambassador/ambassador-specific naming is a general multi-container design pattern rather than a
  dedicated Kubernetes API object, so there's no page beyond this one to cite.

## Setup

```bash
kubectl create ns ambassador
kubectl -n ambassador create deployment backend --image=nginx:1.25-alpine --port=80
kubectl -n ambassador expose deployment backend --port=80
kubectl -n ambassador rollout status deployment backend --timeout=30s
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
  namespace: ambassador
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
  - name: ambassador
    image: alpine/socat
    command: ["sh", "-c", "socat TCP-LISTEN:8080,fork,reuseaddr TCP:backend:80"]
EOF

kubectl -n ambassador wait --for=condition=ready pod/app-with-ambassador --timeout=30s

# the app container only ever addresses localhost — confirmed live
kubectl -n ambassador exec app-with-ambassador -c app -- wget -qO- http://localhost:8080 | head -3
```

The ambassador pattern's whole point: the main container is decoupled from *where* the real
backend lives or how many of it there are — that knowledge lives entirely in the ambassador
container, reachable over `localhost` because all containers in a Pod always share one network
namespace (see `drills/drill.md` 5.4). Swapping the backend, adding retry/circuit-breaker logic, or pointing
at a different environment only ever touches the ambassador container, never the app.

## Cleanup

```bash
kubectl delete ns ambassador
```
