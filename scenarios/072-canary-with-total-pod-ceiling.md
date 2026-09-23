# 072 — Canary deployment under a total-pod ceiling, not just a traffic percentage

**Domain:** Application Deployment · **Difficulty:** Medium

`028` builds both Deployments from scratch and `060` clones a stable Deployment, neither with a
limit on the total. Here the stable Deployment already runs more replicas than the ceiling allows,
so reaching the ratio means **scaling the stable Deployment down**, not just scaling a canary up.
Getting the percentage right while blowing the ceiling (or the other way round) is the trap.

## Task

> In namespace `fuji`, the Service `storefront` sends traffic to the 15 Pods of Deployment
> `storefront`. Start a canary release:
>
> - Create a Deployment `storefront-canary` that is identical to `storefront` apart from its name,
>   with its Pods labelled `track: canary` instead of `track: stable`, so that `storefront` also
>   routes to them.
> - No more than 10 Pods may run in the namespace in total, and 40% of the Service's endpoints
>   must be canary Pods.
>
> Keep the manifest you used for the canary at `~/ckad/072/storefront-canary.yaml`.

## Documentation

What to look up: **Managing Workloads**, canary deployments.
- <https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments>

## Setup

```bash
kubectl create ns fuji

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storefront
  namespace: fuji
spec:
  replicas: 15
  selector:
    matchLabels:
      app: storefront
  template:
    metadata:
      labels:
        app: storefront
        track: stable
    spec:
      containers:
      - image: nginx:1.24-alpine
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: storefront
  namespace: fuji
spec:
  selector:
    app: storefront
  ports:
  - port: 80
EOF

kubectl rollout status deployment/storefront -n fuji --timeout=90s
```

## Solution

**Do the arithmetic before touching any replica count:** 10 Pods in total with 40% canary means 4
canary Pods and 6 stable Pods. So the *stable* Deployment has to come down from 15 to 6. It's easy
to concentrate on the canary's replica count and forget that the stable Deployment is what's over
the ceiling.

Scale the stable Deployment down **first**, so the namespace never goes above 10 Pods:
```bash
kubectl scale deployment storefront -n fuji --replicas=6
kubectl rollout status deployment/storefront -n fuji --timeout=30s
```

Clone the running Deployment (same technique as `060`), then change its name, replica count and
`track` label. The Service's selector (`app: storefront`) must stay the same on both, because that
label is what decides who receives traffic:
```bash
mkdir -p ~/ckad/072
kubectl get deploy storefront -n fuji -o yaml > ~/ckad/072/storefront-canary.yaml
sed -i -e 's/^  name: storefront$/  name: storefront-canary/' \
       -e 's/^  replicas: 6$/  replicas: 4/' \
       -e 's/track: stable/track: canary/' ~/ckad/072/storefront-canary.yaml
kubectl apply -f ~/ckad/072/storefront-canary.yaml
kubectl rollout status deployment/storefront-canary -n fuji --timeout=60s
```
**Faster by hand:** open the dump in `vim` and change the three lines directly (`metadata.name`,
`spec.replicas`, and `track:` under the Pod template's labels). The `sed` line is just a scripted
version of the same three edits. The dump's `status`, `uid` and `resourceVersion` don't need
removing: the API server ignores them when it creates the new Deployment.

Confirm both the ceiling and the ratio. Count the Service's endpoints, not just each Deployment's
`READY` column, since the endpoints are what prove the Service really splits traffic across both:
```bash
kubectl get deploy -n fuji
# storefront          6/6
# storefront-canary   4/4

kubectl get endpoints storefront -n fuji -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w
# 10 — the ceiling
kubectl get pods -n fuji -l track=canary --no-headers | wc -l
# 4 — 40% of 10
```

**Why the order matters:** creating the canary first and scaling stable down afterwards briefly runs
19 Pods. That keeps more capacity serving during the change, but breaks "no more than 10" while it
lasts. If the ceiling were enforced by a ResourceQuota on `pods`, the canary's Pods wouldn't even be
created until stable came down. Scaling stable down first keeps the namespace at or below 10 the
whole time, at the cost of briefly serving from 6 Pods.

## Cleanup

```bash
kubectl delete ns fuji
rm -rf ~/ckad/072
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
