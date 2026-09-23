# 041 — StatefulSet with stable per-Pod network identity via a headless Service

**Domain:** Application Design and Build · **Difficulty:** Medium

StatefulSet isn't named in the curriculum the way Deployment/DaemonSet/CronJob are, but "choose
and use the right workload resource" leaves room for it, and it's worth recognizing on sight.

## Task

> Create a headless Service `web-headless` and a StatefulSet `web` (3 replicas, image
> `nginx:1.25-alpine`) in namespace `statefulns`, such that each Pod gets a stable, individually
> addressable DNS name — confirm Pod `web-1` specifically is reachable by name, not just by label
> selector.

## Documentation

What to look up: **StatefulSets**, and **Headless Services**.
- <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/> — stable network
  identity, ordered deployment/scaling, and the `serviceName` field.
- <https://kubernetes.io/docs/concepts/services-networking/service/#headless-services> — `clusterIP: None`.

## Setup

```bash
kubectl create ns statefulns
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  namespace: statefulns
spec:
  clusterIP: None      # headless — no load-balancing IP, just DNS records per Pod
  selector: {app: web}
  ports:
  - port: 80
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: statefulns
spec:
  serviceName: web-headless   # must match the headless Service's name
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: nginx:1.25-alpine
EOF

kubectl -n statefulns rollout status statefulset web --timeout=60s
kubectl -n statefulns get pods
# web-0, web-1, web-2 — created and numbered in order, unlike a Deployment's random-suffix Pods

kubectl -n statefulns run tmp --restart=Never --rm -i --image=busybox:1.31.0 -- \
  nslookup web-1.web-headless.statefulns.svc.cluster.local
```

Confirmed live: `web-1.<service-name>.<namespace>.svc.cluster.local` resolves directly to that
one Pod's IP — a plain Deployment's Pods have random-suffix names and no individual DNS record at
all, only the Service's single (load-balanced) name. That per-Pod addressability, plus ordered
creation/termination and (with a `volumeClaimTemplates` block, not shown here) a stable PVC per
replica, is the entire reason StatefulSet exists as a separate resource from Deployment.

## Cleanup

```bash
kubectl delete ns statefulns
```
