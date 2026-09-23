# 087 — StatefulSet with `volumeClaimTemplates`: a real PVC per Pod, and ordered shutdown

**Domain:** Application Design and Build · **Difficulty:** Medium

This completes what `041` leaves out: real per-Pod storage through `volumeClaimTemplates`, data
that survives a Pod's deletion because the PVC stays behind, and scale-down that happens one Pod at
a time, highest ordinal first.

## Task

> In namespace `statefulns2`, create StatefulSet `web` with 3 replicas, governed by a headless
> Service `web-headless`. Its container `web` runs `busybox:1.31.0` with `sleep 3600`, and each Pod
> must get its own `100Mi` `ReadWriteOnce` `PersistentVolumeClaim` mounted at `/data`, not one
> shared volume. Write distinct data into each Pod's volume, delete one Pod, and
> confirm its replacement comes back with the *same* data. Then scale down to 1 replica and observe
> the shutdown order.

## Documentation

What to look up: **StatefulSets** — `volumeClaimTemplates`, and ordered Pod termination.
- <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#volume-claim-templates>
- <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#deployment-and-scaling-guarantees> —
  strictly sequential, highest-ordinal-first scale-down, confirmed live in this scenario.

## Setup

```bash
kubectl create ns statefulns2
```

## Solution

A StatefulSet needs a headless Service for its `serviceName`; the per-Pod storage comes from
`volumeClaimTemplates`:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  namespace: statefulns2
spec:
  clusterIP: None
  selector: {app: web}
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: statefulns2
spec:
  serviceName: web-headless
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: busybox:1.31.0
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
EOF

kubectl rollout status statefulset/web -n statefulns2 --timeout=60s
```

`volumeClaimTemplates` is what makes the difference from `041` — the StatefulSet controller creates
one **real** PVC per Pod, named `<template-name>-<pod-name>`, not a single shared volume:
```bash
kubectl get pvc -n statefulns2
# data-web-0   Bound   ...
# data-web-1   Bound   ...
# data-web-2   Bound   ...
```

Write distinct data into each Pod, then delete one and let the StatefulSet controller recreate it:
```bash
for i in 0 1 2; do
  kubectl exec -n statefulns2 web-$i -- sh -c "echo hello-from-web-$i > /data/marker.txt"
done

kubectl delete pod web-1 -n statefulns2
kubectl wait --for=condition=Ready pod/web-1 -n statefulns2 --timeout=30s

kubectl exec -n statefulns2 web-1 -- cat /data/marker.txt
# hello-from-web-1   <- the NEW web-1 Pod, same old data

kubectl get pod web-1 -n statefulns2 -o jsonpath='{.spec.volumes[0].persistentVolumeClaim.claimName}{"\n"}'
# data-web-1   <- reattached to the exact same PVC by name, not a fresh empty one
```
This is the entire reason a StatefulSet exists instead of a Deployment for stateful workloads: the
identity (`web-1` → `data-web-1`) is stable across Pod recreation, where a Deployment's replacement
Pod would get a brand new random name and, with `volumeClaimTemplates`, an entirely different PVC.

Scale down and watch shutdown order:
```bash
kubectl scale statefulset web -n statefulns2 --replicas=1
kubectl get pods -n statefulns2 -w
```
Confirmed live: `web-2` (highest ordinal) starts `Terminating` first and finishes completely before
`web-1` even begins — strictly sequential, one Pod at a time, always highest-ordinal-first. Neither
terminates concurrently, unlike a Deployment's scale-down, which has no ordering guarantee at all.
Each Pod takes about 30 seconds to go, because `sh -c "sleep 3600"` ignores `SIGTERM` and waits out
the grace period, so the whole scale-down takes about a minute. Press Ctrl-C to stop watching.

Check the PVCs after scale-down:
```bash
kubectl get pvc -n statefulns2
# all 3 still Bound — data-web-1 and data-web-2 included, despite only web-0 running now
```

**Lesson:** scaling a StatefulSet down never deletes its PVCs — that's a deliberate safety
guarantee, not an oversight. Scaling back up to 3 later reattaches `web-1` and `web-2` to their
*original* volumes, data intact. A PVC created via `volumeClaimTemplates` only goes away if you
delete it yourself (or delete the whole StatefulSet with the appropriate cascade), never as a side
effect of `kubectl scale`.

## Cleanup

```bash
kubectl delete ns statefulns2
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
