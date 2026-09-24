# 079 — `runAsNonRoot: true` rejects an image that is already non-root

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`051`, `063` and `071` set a specific `runAsUser` UID. `runAsNonRoot` is different: it's a
*boolean check* the kubelet makes against the UID the container would run as, just before starting
it. The check can only pass if the kubelet can *prove* that UID isn't 0, and an image whose `USER`
is a name rather than a number can't be proven either way, even when it's perfectly non-root.

## Task

> In namespace `tungsten`, the bare Pod `session-cache` (image `memcached:1.6-alpine`, no
> controller) is running. The security team wants the kubelet itself to block the container from
> ever being started with UID 0, whatever the image or a later edit says. Add that safeguard, and end
> up with a `session-cache` Pod that has it and is running and serving on port `11211`.

## Documentation

What to look up: **Configure a Security Context**, the `runAsNonRoot` and `runAsUser` fields.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/> — `runAsNonRoot` is
  documented as a check made against the container's user, not as something that changes the user.

## Setup

```bash
kubectl create ns tungsten
kubectl run session-cache -n tungsten --image=memcached:1.6-alpine --port=11211
kubectl wait --for=condition=Ready pod/session-cache -n tungsten --timeout=60s
```

## Solution

Before changing anything, check who the container runs as now:
```bash
kubectl exec session-cache -n tungsten -- id
# uid=11211(memcache) gid=11211(memcache) groups=11211(memcache)
```
It's already non-root, so adding `runAsNonRoot: true` looks like a formality. It's a
`securityContext` change on a bare Pod, though, and those fields can't be changed on a running Pod
(see `071`), so the Pod has to be deleted and created again:
```bash
kubectl delete pod session-cache -n tungsten
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: session-cache
  namespace: tungsten
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: session-cache
    image: memcached:1.6-alpine
    ports:
    - containerPort: 11211
EOF
```
```bash
sleep 5
kubectl get pod session-cache -n tungsten
# STATUS: CreateContainerConfigError

kubectl describe pod session-cache -n tungsten | grep -m1 'non-numeric'
# Error: container has runAsNonRoot and image has non-numeric user (memcache), cannot verify user is non-root ...
```
`CreateContainerConfigError` means the kubelet refused to create the container at all, so this never
shows up as `CrashLoopBackOff` or a climbing restart count. The image's `USER` is the *name*
`memcache`. The kubelet doesn't look inside the image's `/etc/passwd`, so it can't tell whether that
name maps to UID 0, and `runAsNonRoot` fails closed.

The fix is to give the kubelet a number to check: set `runAsUser` to the UID that `id` showed. Using
the image's own UID (rather than any non-zero number) keeps file ownership inside the image
consistent with what the process runs as:
```bash
kubectl delete pod session-cache -n tungsten
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: session-cache
  namespace: tungsten
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 11211
  containers:
  - name: session-cache
    image: memcached:1.6-alpine
    ports:
    - containerPort: 11211
EOF

kubectl wait --for=condition=Ready pod/session-cache -n tungsten --timeout=60s
kubectl exec session-cache -n tungsten -- id
# uid=11211(memcache) gid=11211(memcache) groups=11211(memcache)
```
Check that it actually serves, not just that it started. The Pod IP is enough for a one-off client:
```bash
IP=$(kubectl get pod session-cache -n tungsten -o jsonpath='{.status.podIP}')
kubectl run mc-check -n tungsten --rm -i --restart=Never --image=busybox:1.36 -- \
  sh -c "printf 'stats\r\nquit\r\n' | nc -w 2 $IP 11211 | grep -m1 'STAT pid'"
# STAT pid 1
```

For comparison, all the ways `runAsNonRoot: true` can end, each verified on the same cluster:

| Container's user | Result |
| --- | --- |
| numeric and non-zero (`runAsUser: 11211` here, or an image whose `USER` is a number) | starts |
| root (image with no `USER`, e.g. `busybox`) | `CreateContainerConfigError`: `container has runAsNonRoot and image will run as root` |
| explicit `runAsUser: 0` | `CreateContainerConfigError`: `container's runAsUser breaks non-root policy` |
| a name (image `USER memcache`, as here) | `CreateContainerConfigError`: `image has non-numeric user (...), cannot verify user is non-root` |

**Lesson:** `runAsNonRoot: true` doesn't change who the container runs as; it only refuses to start
it unless the UID is provably non-zero. When the image names its user instead of numbering it, the
guarantee needs a `runAsUser` alongside it, and you find the right number by asking the running
container (`id`) before you delete it.

## Cleanup

```bash
kubectl delete ns tungsten
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
