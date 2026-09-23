# 032 — Startup probe gating a slow app, liveness probe catching a later failure

**Domain:** Application Observability and Maintenance · **Difficulty:** Hard

Scenario 007 covers readiness probes; this one covers the other two.

## Task

> A container is slow to become useful and would fail a normal liveness probe during that startup
> window, incorrectly getting restarted before it ever finishes starting. Use a startup probe to
> protect it during startup, and a liveness probe to catch a genuine failure *after* it's up. Use
> namespace `probens`, Pod name `slow-starter`, image `busybox:1.31.0`.

## Documentation

What to look up: **Configure Liveness, Readiness and Startup Probes**.
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/> —
  the "Protect slow starting containers with startup probes" section explains exactly why startup
  and liveness are separate and how one gates the other.

## Setup

```bash
kubectl create ns probens
```

## Solution

The Pod below demonstrates both phases on its own: it creates a marker file immediately (so
`startupProbe` succeeds fast), stays "alive" for 15s, then removes the marker file to simulate a
genuine later failure (so `livenessProbe` catches it and the container restarts).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-starter
  namespace: probens
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "touch /tmp/live; sleep 15; rm -f /tmp/live; sleep 3600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/live"]
      failureThreshold: 30
      periodSeconds: 1
    livenessProbe:
      exec:
        command: ["cat", "/tmp/live"]
      periodSeconds: 2
      failureThreshold: 1
EOF
```

While `startupProbe` is still running, `livenessProbe` is **not** evaluated at all — that's the
whole point of a startup probe existing separately from liveness. `failureThreshold: 30` ×
`periodSeconds: 1` gives it up to 30s to succeed before the kubelet gives up and restarts the
container (this app succeeds almost immediately, at ~1s).

```bash
kubectl -n probens get pod slow-starter
# READY flips to 1/1 within ~1-2s — startup probe succeeded fast

sleep 35
kubectl -n probens get pod slow-starter
kubectl -n probens describe pod slow-starter | grep -A6 Events:
```

Confirmed live: at ~t=16s the marker file disappears, the liveness probe (checking every 2s) fails
on its next check, and the Events show `Unhealthy` followed by `Killing ... will be restarted` —
`RESTARTS` increments shortly after. Startup and liveness probes can point at the *same* check
(as here) or different ones; what matters is that liveness only starts being enforced once startup
has already succeeded.

## Cleanup

```bash
kubectl delete ns probens
```
