# 010 — ClusterIP Service exposing a new Pod, tested with a temporary curl Pod

**Domain:** Services and Networking · **Difficulty:** Easy

## Task

> The `cedar` team is standing up a billing backend that other workloads in the cluster will call.
> In Namespace `cedar`, run a Pod named `billing-api` from image `nginx:1.27-alpine` with the label
> `app=billing-api`. Put a ClusterIP Service named `billing-svc` in front of it that listens on port
> `8080` and forwards to the container's port `80`. Prove it works by sending an HTTP request to the
> Service from a short-lived Pod, then confirm the request shows up in `billing-api`'s logs.

## Documentation

What to look up: **Service** — the default `ClusterIP` type and selectors.
- <https://kubernetes.io/docs/concepts/services-networking/service/> — "Defining a Service" and
  how `spec.selector` maps to Pod labels.

## Setup

```bash
kubectl create ns cedar
```

Nothing else. The task creates everything itself.

## Solution

```bash
kubectl -n cedar run billing-api --image=nginx:1.27-alpine --labels app=billing-api
kubectl -n cedar wait --for=condition=ready pod/billing-api --timeout=60s

# expose copies the Pod's labels into the Service selector for you
kubectl -n cedar expose pod billing-api --name billing-svc --port 8080 --target-port 80

kubectl -n cedar get pod,svc
kubectl -n cedar get endpoints billing-svc
# ENDPOINTS: <pod-ip>:80. If this is empty, the selector doesn't match the Pod's labels

kubectl run tmp --restart=Never --rm -i --image=nginx:alpine -n cedar -- \
  curl -s -m 5 http://billing-svc:8080
# <!DOCTYPE html> ... <title>Welcome to nginx!</title> ...
# a "couldn't attach to pod/tmp, falling back to streaming logs" warning is harmless: the curl
# finished before kubectl attached, and the output is still printed

kubectl -n cedar logs billing-api | tail -3
# ... "GET / HTTP/1.1" 200 615 "-" "curl/..."
```

The temporary Pod has to run in the same namespace to use the short name `billing-svc`. From
another namespace, use `billing-svc.cedar` or `billing-svc.cedar.svc.cluster.local`.

## Cleanup

```bash
kubectl delete ns cedar
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
