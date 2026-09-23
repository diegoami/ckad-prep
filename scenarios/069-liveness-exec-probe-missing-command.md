# 069 — Liveness exec probe on a container whose command exits immediately

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

`007` and `032` write their probes correctly from the start. Here the probe is fine and the
container command is the bug: it runs once and exits, so the Pod crash-loops before the liveness
probe ever matters.

## Task

> In namespace `annapurna`, the Pod `heartbeat` keeps restarting. Its manifest is at
> `~/ckad/069/heartbeat.yaml`. The liveness probe (it checks that `/tmp/healthy` exists) is
> correct and must stay as it is. Fix the Pod so it stays `Running` with the probe passing, and
> save the fixed manifest back to the same file.

## Documentation

What to look up: **Configure Liveness, Readiness and Startup Probes**, the exec-probe walkthrough.
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command> —
  the page's own example is a container that creates and removes `/tmp/healthy`, the same pattern
  used here.

The docs example uses the image `registry.k8s.io/busybox`, which was published with an old Docker
Schema v1 manifest. containerd v2.1+ refuses to pull it (`media type
"application/vnd.docker.distribution.manifest.v1+prettyjws" is no longer supported`). If you copy
the example on a newer cluster, swap in a tagged `busybox` image such as `busybox:1.31.0`.

## Setup

```bash
kubectl create ns annapurna
mkdir -p ~/ckad/069

cat > ~/ckad/069/heartbeat.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: heartbeat
  name: heartbeat
  namespace: annapurna
spec:
  containers:
  - name: heartbeat
    image: busybox:1.31.0
    args:
    - /bin/sh
    - -c
    - date
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

kubectl apply -f ~/ckad/069/heartbeat.yaml
```

Confirm the "given" broken state:
```bash
sleep 8
kubectl get pod heartbeat -n annapurna
# STATUS: CrashLoopBackOff (or Completed, between restarts)
```
Note *why*: `kubectl describe` shows the container repeatedly `Started` and then gone, with no
`Unhealthy` liveness event. The container exits on its own as soon as `date` finishes printing.
`restartPolicy: Always` (the Pod default) is what restarts it, not the liveness probe, which never
gets to run before the container is already gone.

## Solution

The container needs to do two things it currently doesn't: create `/tmp/healthy` (so the probe has
something to find), and keep running afterwards (so there's a live process for the probe to check).
Append both to the existing command rather than replacing it. Printing `date` once at startup isn't
wrong, it just isn't enough:
```bash
cat > ~/ckad/069/heartbeat.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: heartbeat
  name: heartbeat
  namespace: annapurna
spec:
  containers:
  - name: heartbeat
    image: busybox:1.31.0
    args:
    - /bin/sh
    - -c
    - date; touch /tmp/healthy; while true; do sleep 60; done
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

kubectl delete pod heartbeat -n annapurna --wait=true
kubectl apply -f ~/ckad/069/heartbeat.yaml
```
A running Pod's container command can't be patched in place (it's an immutable field), so recreate
it, the same pattern as the `secret-handler` Pod in `014`. `kubectl replace --force -f
~/ckad/069/heartbeat.yaml` does the delete and create in one command.

Confirm:
```bash
sleep 8
kubectl get pod heartbeat -n annapurna
# READY 1/1, STATUS Running, RESTARTS 0

kubectl describe pod heartbeat -n annapurna | grep -A5 Events:
# no Unhealthy / Killing events — the probe has found /tmp/healthy every time since t=5s
```

**The lesson:** a failing liveness probe and a container that simply exits on its own produce the
*same visible symptom*, `CrashLoopBackOff`. That makes it tempting to suspect the probe config
first (bad path, wrong initial delay). Check whether the container stays up independently of the
probe first (`kubectl describe` events, or temporarily removing the probe) before assuming the
probe is the bug. Here the probe was configured correctly the whole time and the container command
was the problem.

## Cleanup

```bash
kubectl delete ns annapurna
rm -rf ~/ckad/069
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
