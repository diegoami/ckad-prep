# 079 — `runAsNonRoot: true` on an image that defaults to root, and why the fix has two layers

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`051`, `063` and `071` set a specific `runAsUser` UID. `runAsNonRoot` is a *boolean assertion*
checked against whatever UID the image would run as: it fails before the container starts, not as a
runtime crash, and passing it turns out to be only the first of two problems.

## Task

> The platform team in namespace `sicily` wants its front-end Pod `web` to refuse to start if its
> container image would run as root. Create `web` with that guarantee using the stock
> `nginx:1.25-alpine` image and show that it's refused, then end up with a `web` Pod that keeps the
> guarantee and is actually running and serving nginx.

## Documentation

What to look up: **Configure a Security Context** — the `runAsNonRoot` field specifically.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/> — note it's
  documented as a *policy check against the image's configured user*, not a guarantee the app works
  as that user, which is exactly the two-layer gotcha this scenario walks through.

## Setup

```bash
kubectl create ns sicily
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: sicily
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: web
    image: nginx:1.25-alpine
EOF
```

Confirm the rejection — and notice it's a distinct failure mode from a crash:
```bash
kubectl get pod web -n sicily
# STATUS: CreateContainerConfigError

kubectl describe pod web -n sicily | tail -3
# Warning  Failed  ...  container has runAsNonRoot and image will run as root
```
`CreateContainerConfigError` means the kubelet refused to even create the container process — it
checks the image's configured user at container-creation time, so this never shows up as
`CrashLoopBackOff` or a climbing restart count. `nginx:1.25-alpine`'s image defaults to `USER root`
(needed so its entrypoint can bind port 80 and set up its cache directories before dropping
privileges internally), which is exactly what `runAsNonRoot: true` refuses to run at all.

**The obvious next move — just add an arbitrary non-root `runAsUser` — passes the gate but doesn't
actually work:**
```bash
kubectl delete pod web -n sicily
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: sicily
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
  containers:
  - name: web
    image: nginx:1.25-alpine
EOF
```
```bash
kubectl get pod web -n sicily
# STATUS: Error / CrashLoopBackOff — different failure than before, container DID start this time

kubectl logs web -n sicily
# nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```
The `runAsNonRoot` check only cares that the UID is non-zero — it has no idea whether that
UID can actually write to the directories the image's entrypoint needs. UID `101` satisfies the
*policy*, then immediately fails at *runtime* because the stock `nginx` image's filesystem
permissions were never set up for an arbitrary non-root user, only for its internal root→nginx-user
privilege drop.

The real fix is an image actually built to run unprivileged from the start, not a UID bolted onto
one that wasn't:
```bash
kubectl delete pod web -n sicily
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: sicily
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: web
    image: nginxinc/nginx-unprivileged:1.25-alpine
EOF
```
```bash
kubectl get pod web -n sicily
# READY 1/1, STATUS Running

kubectl logs web -n sicily | tail -3
# start worker process ...  — genuinely serving, not just passing the runAsNonRoot check
```

**Lesson:** `runAsNonRoot: true` is a policy check, not a guarantee the app works — satisfying it
(image running as *some* non-zero UID) and the app *functioning* as that UID are two separate
problems with two separate fixes. An arbitrary `runAsUser` addresses only the first; the image
itself has to be built to tolerate running unprivileged for the second.

## Cleanup

```bash
kubectl delete ns sicily
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
