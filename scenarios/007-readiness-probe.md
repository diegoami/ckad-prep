# 007 — Pod with an exec readiness probe

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

## Task

> The `sparrow` team needs a Pod that only reports Ready once it has finished warming up. In
> Namespace `sparrow`, create a Pod named `cache-warmer` from image `busybox:1.36`. Its container
> runs `touch /tmp/cache-ready && sleep 1d`: it creates a marker file and then stays idle.
>
> Give the container a readiness probe that runs `cat /tmp/cache-ready`, so the container counts
> as ready only while that file exists. The first check should happen 4 seconds after the
> container starts, then every 8 seconds. Create the Pod and confirm it becomes Ready.

## Documentation

What to look up: **Configure Liveness, Readiness and Startup Probes** — the exec-probe section.
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/> —
  `exec.command`, `initialDelaySeconds`, `periodSeconds` field meanings.

## Setup

```bash
kubectl create ns sparrow
mkdir -p ~/ckad/007
```

## Solution

```bash
kubectl -n sparrow run cache-warmer --image=busybox:1.36 --dry-run=client -o yaml \
  --command -- sh -c "touch /tmp/cache-ready && sleep 1d" > ~/ckad/007/cache-warmer.yaml
```

Add the readiness probe to the container:

```yaml
spec:
  containers:
  - name: cache-warmer
    image: busybox:1.36
    command: ["sh", "-c", "touch /tmp/cache-ready && sleep 1d"]
    readinessProbe:                 # add
      exec:                         # add
        command: ["cat", "/tmp/cache-ready"]   # add
      initialDelaySeconds: 4        # add
      periodSeconds: 8              # add
```

```bash
kubectl apply -f ~/ckad/007/cache-warmer.yaml
kubectl -n sparrow wait --for=condition=ready pod/cache-warmer --timeout=60s
# pod/cache-warmer condition met
kubectl -n sparrow get pod cache-warmer
# NAME           READY   STATUS    RESTARTS   AGE
# cache-warmer   1/1     Running   0          10s
```

Watching with `kubectl -n sparrow get pod cache-warmer -w`, you see `0/1 Running` for the first
few seconds, then `1/1` once the first probe succeeds. `STATUS` stays `Running` the whole time:
readiness only affects the `READY` column and whether the Pod receives Service traffic. A failing
readiness probe never restarts the container; that's what a liveness probe does.

To check that the probe really is gating readiness, delete the file and watch the Pod drop back
to not-ready:

```bash
kubectl -n sparrow exec cache-warmer -- rm /tmp/cache-ready
sleep 30
kubectl -n sparrow get pod cache-warmer
# NAME           READY   STATUS    RESTARTS   AGE
# cache-warmer   0/1     Running   0          45s
```

It doesn't flip after a single failed check. `failureThreshold` defaults to 3, so the Pod is marked
unready only after three consecutive failures, up to 3 × 8 s = 24 s here.

## Cleanup

```bash
kubectl delete ns sparrow
rm -rf ~/ckad/007
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
