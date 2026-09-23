# 035 — Autoscale a Deployment on CPU utilization

**Domain:** Application Observability and Maintenance · **Difficulty:** Medium

## Task

> Deployment `web` in namespace `hpans` needs to scale itself between 1 and 4 replicas based on CPU
> usage, targeting 50% utilization. It has no resource requests set yet — fix that first, since HPA
> can't compute a percentage without a baseline to divide by.

## Documentation

What to look up: **HorizontalPodAutoscaler Walkthrough**.
- <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/> — how target CPU
  utilization maps to desired replica count, and `kubectl autoscale` vs. a full HPA manifest.

## Setup

```bash
kubectl create ns hpans
kubectl -n hpans create deployment web --image=nginx:1.25-alpine
```

## Solution

```bash
# HPA needs a CPU *request* to measure utilization against — without one, TARGETS stays <unknown> forever
kubectl -n hpans set resources deployment web --requests=cpu=50m --limits=cpu=100m

kubectl -n hpans autoscale deployment web --cpu=50% --min=1 --max=4
```

`--cpu-percent` (older flag, still findable in `-h` output and older docs) is deprecated in favor
of `--cpu`, which accepts either a percentage (`50%`) or an absolute milliCPU quantity (`500m`) —
`--cpu-percent` still works but prints a deprecation warning, `--cpu=50%` doesn't. If your
`kubectl` is older and rejects `--cpu` as an unknown flag, use `--cpu-percent=50` instead.

```bash
kubectl -n hpans get hpa
# TARGETS shows cpu: <unknown>/50% until metrics-server has scraped at least once — needs
# metrics-server installed (kind doesn't ship it — see guide/practice-cluster.md) before it
# resolves to a real number
```

`kubectl autoscale` creates a `HorizontalPodAutoscaler` object targeting the Deployment by name —
`kubectl get hpa` and `kubectl describe hpa web` are the equivalent of `rollout status` for
checking in on it afterward.

## Cleanup

```bash
kubectl delete ns hpans
```
