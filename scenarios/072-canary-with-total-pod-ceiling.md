# 072 — Canary release inside a Pod quota the stable version already exceeds

**Domain:** Application Deployment · **Difficulty:** Medium

`028` builds both Deployments from scratch and `060` clones a stable Deployment, neither with a
limit on the total. Here a ResourceQuota caps the namespace's Pod count, and the stable Deployment
already runs more replicas than the cap allows, so reaching the ratio means **scaling the stable
Deployment down**, not just scaling a canary up. Getting the percentage right while ignoring the
cap (or the other way round) is the trap.

## Task

> The Vela mapping team serves map tiles from Deployment `tile-server` in namespace `vela`, behind
> the Service `tile-server`. Version `nginx:1.27-alpine` is ready for a canary release:
>
> - Create a Deployment `tile-server-canary` that matches `tile-server` except for its name, the
>   image `nginx:1.27-alpine`, and the label `release: canary` in place of `release: stable`. The
>   Service `tile-server` must send traffic to both.
> - The ResourceQuota `pod-ceiling` in `vela` allows at most 8 Pods. When you're done, the
>   namespace must be using all 8, and exactly a quarter of the Service's endpoints must be canary
>   Pods.
>
> Keep the manifest you used for the canary at `~/ckad/072/tile-server-canary.yaml`.

## Documentation

What to look up: **Managing Workloads**, canary deployments, plus **Resource Quotas**.
- <https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments>
- <https://kubernetes.io/docs/concepts/policy/resource-quotas/#quota-on-object-count> — the `pods`
  count quota. The top of the same page notes that changes to a quota don't affect resources that
  already exist: it refuses new Pods but never evicts running ones.

## Setup

```bash
kubectl create ns vela

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tile-server
  namespace: vela
spec:
  replicas: 10
  selector:
    matchLabels:
      app: tile-server
      release: stable
  template:
    metadata:
      labels:
        app: tile-server
        release: stable
    spec:
      containers:
      - name: tiles
        image: nginx:1.25-alpine
---
apiVersion: v1
kind: Service
metadata:
  name: tile-server
  namespace: vela
spec:
  selector:
    app: tile-server
  ports:
  - port: 8080
    targetPort: 80
EOF
kubectl rollout status deployment/tile-server -n vela --timeout=90s

kubectl create quota pod-ceiling -n vela --hard=pods=8
```

## Solution

Look at the starting point first:
```bash
kubectl get quota pod-ceiling -n vela
# NAME          AGE   REQUEST     LIMIT
# pod-ceiling   5s    pods: 10/8
```
The quota was created after the stable Pods were already running, so it reports 10 used out of 8
allowed. A quota never removes existing Pods; it only refuses new ones while the namespace is at
or over the limit.

**Do the arithmetic before touching any replica count:** 8 Pods in total with a quarter canary
means 2 canary Pods and 6 stable Pods. So the *stable* Deployment has to come down from 10 to 6.
It's easy to concentrate on the canary's replica count and forget that the stable Deployment is what
breaks the cap.

Scale the stable Deployment down **first**, to make room:
```bash
kubectl scale deployment tile-server -n vela --replicas=6
kubectl rollout status deployment/tile-server -n vela --timeout=30s
```

Clone the running Deployment (same technique as `060`), then change its name, replica count, image
and `release` label. The label appears twice in the dump, in `spec.selector.matchLabels` and in the
Pod template, and both must change, or the API server rejects the Deployment because its selector
doesn't match its template. The Service's selector (`app: tile-server`) stays the same on both,
because that label is what decides who receives traffic:
```bash
mkdir -p ~/ckad/072
kubectl get deploy tile-server -n vela -o yaml > ~/ckad/072/tile-server-canary.yaml
sed -i -e 's/^  name: tile-server$/  name: tile-server-canary/' \
       -e 's/^  replicas: 6$/  replicas: 2/' \
       -e 's/release: stable/release: canary/' \
       -e 's/image: nginx:1.25-alpine/image: nginx:1.27-alpine/' ~/ckad/072/tile-server-canary.yaml
kubectl apply -f ~/ckad/072/tile-server-canary.yaml
kubectl rollout status deployment/tile-server-canary -n vela --timeout=60s
```
**Faster by hand:** open the dump in `vim` and change the lines directly (`metadata.name`,
`spec.replicas`, both `release:` labels, and `image:`). The `sed` line is just a scripted version of
the same edits. The dump's `status`, `uid` and `resourceVersion` don't need removing: the API
server ignores them when it creates the new Deployment.

Even in the right order, `rollout status` on the canary may sit at `0 out of 2 new replicas` for a
few seconds, with the same `exceeded quota ... used: pods=10` events as below. The quota's usage
count is recalculated asynchronously and the stable Pods that are shutting down still count until
they're gone, so the first create attempts can be refused. The ReplicaSet retries on its own and the
rollout completes; there's nothing to fix.

Confirm the cap and the ratio. Count the Service's endpoints, not just each Deployment's `READY`
column, since the endpoints are what prove the Service really splits traffic across both:
```bash
kubectl get deploy -n vela
# NAME                 READY   UP-TO-DATE   AVAILABLE
# tile-server          6/6     6            6
# tile-server-canary   2/2     2            2

kubectl get quota pod-ceiling -n vela
# pod-ceiling   pods: 8/8

kubectl get endpoints tile-server -n vela -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w
# 8
kubectl get pods -n vela -l release=canary --no-headers | wc -l
# 2 — a quarter of 8
```

**Why the order matters:** create the canary while the stable Deployment still has 10 Pods and the
quota refuses every canary Pod. The Deployment is created, but its ReplicaSet is stuck at `0/2`:
```
kubectl get deploy tile-server-canary -n vela
# tile-server-canary   0/2     0            0
kubectl get events -n vela --field-selector reason=FailedCreate
# ... Error creating: pods "tile-server-canary-..." is forbidden: exceeded quota: pod-ceiling,
#     requested: pods=1, used: pods=10, limited: pods=8
```
The ReplicaSet retries with a back-off, so the canary Pods do appear some time after you scale the
stable Deployment down, but until then it looks as if the canary is broken. Without a quota
nothing stops you, and creating the canary first would briefly run 12 Pods, over the ceiling
the task sets. Scaling stable down first stays within the cap the whole time, at the cost of
briefly serving from 6 Pods.

## Cleanup

```bash
kubectl delete ns vela
rm -rf ~/ckad/072
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
