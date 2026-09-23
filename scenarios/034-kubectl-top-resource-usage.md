# 034 — Find the pod using the most CPU with `kubectl top`

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

## Task

> Namespace `topns` has a Deployment `stress` under CPU load. Identify which of its Pods is using
> the most CPU right now, using `kubectl` directly — no external monitoring stack.

## Documentation

What to look up: **Resource metrics pipeline**.
- <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/> — what
  `kubectl top` actually reads (the Metrics API, backed by `metrics-server`) and why it's empty/errors
  without that component installed.

## Setup

```bash
# metrics-server isn't preinstalled on kind — see guide/practice-cluster.md
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl rollout status deployment metrics-server -n kube-system --timeout=60s
sleep 15

kubectl create ns topns
kubectl -n topns create deployment stress --image=polinux/stress --replicas=1 -- stress --cpu 1 --timeout 120s
kubectl -n topns wait --for=condition=ready pod -l app=stress --timeout=60s
sleep 20
```

## Solution

Without metrics-server, `kubectl top` fails with `Metrics API not available`. The
`--kubelet-insecure-tls` patch is needed because kind's kubelets serve self-signed certificates;
the first scrape completes roughly 15s after the rollout. The Setup's final `sleep 20` lets the
stress Pod burn CPU for a few scrape intervals before you measure.

```bash
kubectl -n topns top pod --sort-by=cpu
# NAME                      CPU(cores)   MEMORY(bytes)
# stress-6b8859f499-k9jwv   991m         0Mi
```

`--sort-by=cpu` (or `=memory`) sorts descending by that column — the top row is always the answer
to "which pod is using the most." Works cluster-wide too: `kubectl top pod -A --sort-by=cpu`.
`kubectl top node` is the equivalent one level up, for "which *node* is under pressure" instead of
which pod.

## Cleanup

```bash
kubectl delete ns topns
```
