# 075 — Size a Pod's limit at half the namespace's ResourceQuota ceiling, then find the heaviest Pod

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`052` and `058` fit a Deployment's requests and limits into an existing quota. Here it's a bare Pod,
and the limit is given as arithmetic on the quota itself ("half the namespace's memory limit")
rather than a number. A second, unrelated `kubectl top` check is bundled into the same task, the
way exam questions often combine two loosely related asks.

## Task

> Namespace `cotopaxi` has a ResourceQuota. Create a Pod `report-api` (image `nginx:1.25-alpine`)
> that requests at least `200m` CPU and `128Mi` memory, with a memory **limit** of exactly half the
> namespace's `limits.memory` quota.
>
> Then find which Pod in `cotopaxi` is currently using the most memory, and write its name to
> `~/ckad/075/heaviest-pod.txt`.

## Documentation

What to look up: **Resource Quotas**, plus **Resource metrics pipeline** for `kubectl top`.
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/>

## Setup

Needs metrics-server on your local kind cluster (see `guide/practice-cluster.md`).

```bash
kubectl create ns cotopaxi

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: cotopaxi
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    limits.cpu: "2"
    limits.memory: 680Mi
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-shipper
  namespace: cotopaxi
spec:
  containers:
  - name: log-shipper
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 50m, memory: 32Mi}
      limits: {cpu: 100m, memory: 64Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: cache-warmer
  namespace: cotopaxi
spec:
  containers:
  - name: cache-warmer
    image: polinux/stress
    command: ["stress", "--vm", "1", "--vm-bytes", "150M", "--vm-keep", "--timeout", "3600s"]
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits: {cpu: 300m, memory: 200Mi}
EOF

kubectl wait --for=condition=Ready pod/log-shipper pod/cache-warmer -n cotopaxi --timeout=90s
```

`--vm-keep` makes `stress` hold its 150M instead of allocating and freeing it in a loop, so
`cache-warmer`'s memory reading stays steady from one metrics sample to the next.

## Solution

Read the quota's actual ceiling before doing any arithmetic on it. Don't assume a round number:
```bash
kubectl describe resourcequota compute-quota -n cotopaxi | grep -A6 "Resource "
# limits.memory    ...    680Mi   <- the "Hard" column is what matters here
```
Half of `680Mi` is `340Mi`. That's the new container's memory *limit*, not its request (the task
gives the requests directly: `128Mi` and at least `200m` CPU):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: report-api
  namespace: cotopaxi
spec:
  containers:
  - name: report-api
    image: nginx:1.25-alpine
    resources:
      requests:
        memory: "128Mi"
        cpu: "200m"
      limits:
        memory: "340Mi"
        cpu: "500m"
EOF

kubectl wait --for=condition=Ready pod/report-api -n cotopaxi --timeout=30s
```
The task doesn't mention a CPU limit, but the quota sets `limits.cpu`. That means (same rule as
`052`) *every* Pod in the namespace must declare a CPU limit or be rejected outright, even one whose
CPU was never mentioned in the task. `500m` here is an arbitrary value inside the remaining
headroom, not derived from anything in the task.

Find the heaviest consumer. This needs metrics-server running (`kubectl top nodes` failing with
`error: Metrics API not available` means it isn't installed):
```bash
kubectl top pods -n cotopaxi --sort-by=memory
# NAME           CPU(cores)   MEMORY(bytes)
# cache-warmer   250m         150Mi
# report-api     0m           22Mi
# log-shipper    0m           0Mi

mkdir -p ~/ckad/075
kubectl top pods -n cotopaxi --sort-by=memory --no-headers | head -1 | awk '{print $1}' > ~/ckad/075/heaviest-pod.txt
cat ~/ckad/075/heaviest-pod.txt
# cache-warmer
```
`--sort-by=memory` (or `cpu`) sorts the output for you, so there's no eyeballing an unsorted list.
`cache-warmer` is the answer because it has the highest actual usage, not the highest *limit*:
`kubectl top` reports live consumption, which is a different number from anything in the Pod spec
or the ResourceQuota.

**Note:** `kubectl top` can take ~15–60s after Pods start before the first metrics are available.
`error: metrics not available yet`, or a Pod missing from the list, straight after creating Pods
usually means "too soon", not "broken".

## Cleanup

```bash
kubectl delete ns cotopaxi
rm -rf ~/ckad/075
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
