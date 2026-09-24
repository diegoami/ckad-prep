# 075 — Size a Pod to the quota headroom that's left, then find the top CPU consumer

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`052` and `058` fit a Deployment's requests and limits into an existing quota. Here it's a bare Pod
in a namespace that's already partly used, and its memory limit is defined by the quota's
*remaining headroom* rather than a number, so you have to read the `Used` column as well as `Hard`.
A second, loosely related `kubectl top` check is bundled into the same task, the way exam questions
often combine two asks.

## Task

> The `nickel` team's namespace `nickel` has a ResourceQuota and three Pods already running. Create
> a Pod `invoice-renderer` (image `httpd:2.4.62-alpine`) that requests `150m` CPU and `96Mi`
> memory, with a CPU limit of three times its CPU request, and a memory limit that uses up all of
> the `limits.memory` headroom still left in the quota.
>
> Then find which Pod in `nickel` is currently using the most CPU, and write its name to
> `~/ckad/075/top-cpu-pod.txt`.

## Documentation

What to look up: **Resource Quotas**, plus **Resource metrics pipeline** for `kubectl top`.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/>

## Setup

Needs metrics-server on your local kind cluster (see `guide/practice-cluster.md`).

```bash
kubectl create ns nickel

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: nickel
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 1600Mi
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ledger-sync
  namespace: nickel
spec:
  containers:
  - name: ledger-sync
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 50m, memory: 64Mi}
      limits: {cpu: 250m, memory: 256Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: fx-rates
  namespace: nickel
spec:
  containers:
  - name: fx-rates
    image: polinux/stress
    command: ["stress", "--cpu", "1", "--timeout", "3600s"]
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits: {cpu: 400m, memory: 224Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: pdf-cache
  namespace: nickel
spec:
  containers:
  - name: pdf-cache
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 100m, memory: 112Mi}
      limits: {cpu: 600m, memory: 544Mi}
EOF

kubectl wait --for=condition=Ready pod/ledger-sync pod/fx-rates pod/pdf-cache -n nickel --timeout=90s
```

## Solution

"Headroom" is `Hard` minus `Used`, so read both columns rather than assuming the namespace is
empty:
```bash
kubectl describe resourcequota team-quota -n nickel
# Resource         Used   Hard
# --------         ----   ----
# limits.cpu       1250m  2
# limits.memory    1Gi    1600Mi
# requests.cpu     250m   2
# requests.memory  240Mi  1Gi
```
`Used` is shown in the largest unit that divides evenly, so `1Gi` here is `1024Mi` (256 + 224 +
544). The headroom is `1600Mi - 1024Mi = 576Mi`. The CPU limit is `3 × 150m = 450m`, which fits in
the `750m` of `limits.cpu` headroom:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: invoice-renderer
  namespace: nickel
spec:
  containers:
  - name: invoice-renderer
    image: httpd:2.4.62-alpine
    resources:
      requests:
        cpu: "150m"
        memory: "96Mi"
      limits:
        cpu: "450m"
        memory: "576Mi"
EOF

kubectl wait --for=condition=Ready pod/invoice-renderer -n nickel --timeout=60s
kubectl describe resourcequota team-quota -n nickel | grep limits.memory
# limits.memory    1600Mi  1600Mi
```
A limit exactly equal to the headroom is admitted: the quota check is "Used + new ≤ Hard", not
"<". The side effect is that `limits.memory` is now full, so the next Pod with *any* memory limit
is rejected with `exceeded quota: team-quota ... limits.memory`. And because the quota sets
`limits.cpu` and `limits.memory`, every Pod in the namespace must declare both limits or be
rejected outright (same rule as `052`).

Find the top CPU consumer. This needs metrics-server running (`kubectl top nodes` failing with
`error: Metrics API not available` means it isn't installed):
```bash
kubectl top pods -n nickel --sort-by=cpu
# NAME               CPU(cores)   MEMORY(bytes)
# fx-rates           ~400m        0Mi
# invoice-renderer   4m           5Mi     <- may be missing for the first minute
# ledger-sync        0m           0Mi
# pdf-cache          0m           0Mi

mkdir -p ~/ckad/075
kubectl top pods -n nickel --sort-by=cpu --no-headers | head -1 | awk '{print $1}' > ~/ckad/075/top-cpu-pod.txt
cat ~/ckad/075/top-cpu-pod.txt
# fx-rates
```
`--sort-by=cpu` (or `memory`) sorts the output for you, so there's no eyeballing an unsorted list.
`pdf-cache` has the highest CPU *limit* (`600m`) but sits idle. `kubectl top` reports live
consumption, which is a different number from anything in the Pod spec or the ResourceQuota, and
`fx-rates` sits at its own `400m` limit because it's being throttled there.

**Note:** `kubectl top` can take ~15–60s after Pods start before the first metrics are available.
`error: metrics not available yet`, or a Pod missing from the list, straight after creating Pods
usually means "too soon", not "broken".

## Cleanup

```bash
kubectl delete ns nickel
rm -rf ~/ckad/075
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
