# 065 — Add an HTTP readiness probe to an already-running Deployment

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

`007` writes a new Pod with an exec probe from scratch. This is the more common exam shape: a
Deployment is already running without a probe, and you add an `httpGet` readiness probe to the
existing container.

## Task

> Deployment `web` in namespace `rainier` is running `nginx` with no readiness probe configured —
> its Pods report `1/1 Ready` the instant the container starts, even before nginx has actually
> finished starting up. Add an HTTP readiness probe checking `GET /` on port 80, with a 3 second
> initial delay and a 5 second check interval, and confirm a fresh Pod briefly shows `0/1` before
> flipping to `1/1`.

## Documentation

What to look up: **Configure Liveness, Readiness and Startup Probes** — the `httpGet` section.
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes>

## Setup

```bash
kubectl create ns rainier
kubectl create deployment web -n rainier --image=nginx:1.25-alpine --port=80
kubectl rollout status deployment/web -n rainier --timeout=30s
```

## Solution

There's no `kubectl set readinessProbe` — patch the container directly. Name the container
explicitly (`kubectl create deployment` names the container after the image, `nginx` here) so the
strategic merge patch lands on the right one:
```bash
kubectl patch deployment web -n rainier -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":3,"periodSeconds":5}}]}}}}'
```
**Faster by hand:** `kubectl edit deployment web -n rainier`, add the whole `readinessProbe:`
block under the container directly, save — no container-name targeting needed since you're just
typing it in place.
```bash
kubectl rollout status deployment/web -n rainier --timeout=30s
```

Confirm the probe is actually attached:
```bash
kubectl get deploy web -n rainier -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'
# {"httpGet":{"path":"/","port":80,"scheme":"HTTP"},"initialDelaySeconds":3,"periodSeconds":5,...}
```

Force a fresh Pod and watch `READY` transition — this is the part that actually proves the probe is
doing something, not just present in the spec:
```bash
POD=$(kubectl get pods -n rainier -l app=web --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')
kubectl delete pod "$POD" -n rainier --grace-period=0 --force

kubectl get pods -n rainier -w
# web-...   0/1   Running   0   1s   <- container's up, probe hasn't passed yet
# web-...   1/1   Running   0   7s   <- flips once the first successful check lands (~3-8s in)
```
`Ctrl-C` once it shows `1/1` — the window is short (default `nginx` starts almost instantly), so
watching right after a fresh Pod creation is the only reliable way to actually see the `0/1` state
rather than just trusting the spec looks right. Pick the newest Pod: straight after the rollout
the old Pod can still be `Terminating`, and `{.items[0]}` may pick that one instead, so the Pod you
delete isn't the one serving.

## Cleanup

```bash
kubectl delete ns rainier
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
