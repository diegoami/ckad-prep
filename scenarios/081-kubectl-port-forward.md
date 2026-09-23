# 081 — Reach a Pod (or Service) directly with `kubectl port-forward`, no Service required

**Domain:** Services and Networking · **Difficulty:** Easy

The other connectivity scenarios (`010`, `018`, `049`, `055`, `056`) test through a temporary
in-cluster curl Pod or an Ingress. `port-forward` reaches a Pod **directly from your own shell**
without creating anything in the cluster, the fastest way to poke at a single Pod.

## Task

> Pod `web` (namespace `forwardns`) runs nginx on port 80, with no Service. Reach it from your local
> shell on local port `8888`, without creating a Service or a temporary debug Pod.

## Documentation

What to look up: **Use Port Forwarding to Access Applications in a Cluster**.
- <https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/> —
  the `<local-port>:<pod-port>` syntax, and that it works against a Pod, Deployment, or Service alike.

## Setup

```bash
kubectl create ns forwardns
kubectl run web --image=nginx:1.25-alpine -n forwardns
kubectl wait --for=condition=Ready pod/web -n forwardns --timeout=30s
```

## Solution

```bash
kubectl port-forward pod/web -n forwardns 8888:80 &
```
Runs in the foreground by default — backgrounding it with `&` (or a separate terminal on the real
exam) is what lets you keep issuing commands. `<local-port>:<pod-port>` is the mapping direction —
easy to get backwards under pressure. Wait for the `Forwarding from 127.0.0.1:8888 -> 80` line
before curling; a request sent before the tunnel is up fails with `HTTP 000`.

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8888/
# HTTP 200

curl -s http://localhost:8888/ | head -3
# <!DOCTYPE html> ...
```

Stop it when done — it's a live foreground process, not a cluster object, so there's nothing to
`kubectl delete`:
```bash
kill %1   # or fg then Ctrl-C, if not backgrounded
```

`port-forward` also targets a Service directly (forwards to one of its endpoint Pods, chosen the
same way the Service would route) — useful when a Service already exists but you don't want to rely
on cluster-internal DNS or a NodePort to test it:
```bash
kubectl expose pod web -n forwardns --port=80
kubectl port-forward svc/web -n forwardns 8889:80 &
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8889/
# HTTP 200
kill %1
```

**Why this matters beyond convenience:** `port-forward` tunnels through the existing connection to
the API server (the same one `kubectl exec`/`kubectl logs` use) rather than depending on any
cluster-specific network path — it works identically whether or not a NodePort/Ingress/LoadBalancer
is reachable from wherever `kubectl` is running, which makes it the most portable way to sanity-check
a Pod is actually serving traffic before spending time debugging a Service or Ingress on top of it.

## Cleanup

```bash
kubectl delete ns forwardns
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
