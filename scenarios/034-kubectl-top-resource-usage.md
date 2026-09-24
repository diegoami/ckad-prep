# 034 — Flag the Pod holding the most memory with `kubectl top`

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

## Task

> The Pavo team's image pipeline runs in namespace `pavo` as three Deployments: `thumbnailer`,
> `exif-scanner` and `watermarker`. The node is short on memory and someone has to look at the
> worst offender first. Using `kubectl` only (no monitoring stack), find the Pod in `pavo` with the
> highest **memory** usage right now and add the label `memory-review=pending` to that Pod and to
> no other.

## Documentation

What to look up: **Resource metrics pipeline**.
- <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/> — what
  `kubectl top` actually reads (the Metrics API, backed by `metrics-server`) and why it's empty/errors
  without that component installed.

## Setup

```bash
# metrics-server isn't preinstalled on kind — see guide/practice-cluster.md
if ! kubectl get deployment metrics-server -n kube-system >/dev/null 2>&1; then
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  kubectl patch deployment metrics-server -n kube-system --type=json \
    -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
fi
kubectl rollout status deployment metrics-server -n kube-system --timeout=90s

kubectl create ns pavo
kubectl -n pavo create deployment thumbnailer --image=polinux/stress --replicas=2 -- stress --vm 1 --vm-bytes 48M --vm-hang 0
kubectl -n pavo create deployment exif-scanner --image=polinux/stress -- stress --vm 1 --vm-bytes 160M --vm-hang 0
kubectl -n pavo create deployment watermarker --image=polinux/stress -- stress --cpu 1
kubectl -n pavo wait --for=condition=Available deployment --all --timeout=90s
sleep 45
```

## Solution

Without metrics-server, `kubectl top` fails with `Metrics API not available`. The
`--kubelet-insecure-tls` patch is needed because kind's kubelets serve self-signed certificates.
metrics-server only has numbers for a Pod after it has scraped it a couple of times, so the Setup's
final `sleep 45` gives the new Pods time to show up (right after they start, `kubectl top` can
report `metrics not available yet`).

```bash
kubectl -n pavo top pod --sort-by=memory
# NAME                            CPU(cores)   MEMORY(bytes)
# exif-scanner-779bc567d5-rmr6d   0m           160Mi
# thumbnailer-65bf9dbf8-brcjh     0m           48Mi
# thumbnailer-65bf9dbf8-7jv58     0m           48Mi
# watermarker-cf4bf558f-vx6wh     1066m        0Mi
```

`--sort-by=memory` (or `=cpu`) sorts descending by that column, so the top row is the answer. Label
it without retyping the generated name:
```bash
POD=$(kubectl -n pavo top pod --sort-by=memory --no-headers | head -1 | awk '{print $1}')
kubectl -n pavo label pod "$POD" memory-review=pending

kubectl -n pavo get pods -l memory-review=pending
# exactly one Pod: the exif-scanner one
```

**The trap:** "most resources" questions are often about CPU, and `--sort-by=cpu` is the version
people remember. Here it puts `watermarker` on top, burning a full core with next to no memory:
```bash
kubectl -n pavo top pod --sort-by=cpu --no-headers | head -1
# watermarker-cf4bf558f-vx6wh   1066m   0Mi    <- the busiest Pod, but not the one asked for
```
Read which column the task names before you sort. Also note that `kubectl top` shows current
*usage*. The `requests`/`limits` in `kubectl describe pod` are reservations and caps, not what the
Pod is actually consuming, so they can't answer this question.

Two related views: `kubectl top pod --containers` splits the numbers per container (useful when a
multi-container Pod is the heavy one and the task asks which container), and `kubectl top node`
is the same idea one level up, for "which *node* is under pressure".

## Cleanup

```bash
kubectl delete ns pavo
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
