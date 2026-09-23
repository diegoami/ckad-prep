# 085 — The three QoS classes, and what an OOM kill actually looks like on this cluster

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

The quota and limit scenarios (`052`, `058`, `075`, `082`) are about satisfying a constraint.
This one is about the QoS class Kubernetes derives from those same `resources:` fields, and about
recognising a container that was actually killed for exceeding its memory limit.

## Task

> In namespace `qosns`, create three Pods `guaranteed-pod`, `burstable-pod` and `besteffort-pod`
> (image `busybox:1.31.0`, command `sleep 3600`) that end up in the QoS classes `Guaranteed`,
> `Burstable` and `BestEffort` respectively, purely by how their `resources:` block is written.
> Confirm each class with `kubectl get pod -o jsonpath`. Then create Pod `oom-pod` running
> `polinux/stress` with `stress --vm 1 --vm-bytes 200M --vm-hang 1` and a memory limit of `64Mi`,
> and diagnose why it keeps restarting.

## Documentation

What to look up: **Configure Quality of Service for Pods**.
- <https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/> — the exact
  rule for each of `Guaranteed`/`Burstable`/`BestEffort`, and how OOM-kill scoring uses QoS class
  (lower-priority/BestEffort Pods get killed first under node memory pressure).

## Setup

```bash
kubectl create ns qosns
```

## Solution

The class follows from the `resources:` block alone:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: qosns
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits: {cpu: 100m, memory: 64Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  namespace: qosns
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits: {cpu: 200m, memory: 128Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
  namespace: qosns
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl wait --for=condition=Ready pod --all -n qosns --timeout=30s
```

```bash
for p in guaranteed-pod burstable-pod besteffort-pod; do
  echo "$p: $(kubectl get pod $p -n qosns -o jsonpath='{.status.qosClass}')"
done
# guaranteed-pod: Guaranteed
# burstable-pod: Burstable
# besteffort-pod: BestEffort
```
The rule behind all three: `Guaranteed` needs **every** container's `requests` to exactly equal its
`limits`, for **both** CPU and memory — miss just the CPU limit and it downgrades to `Burstable`
even if memory matches exactly. `BestEffort` needs **no** `resources:` block at all, on **any**
container — one container with even a bare `requests.cpu` bumps the whole Pod to `Burstable`.
`Burstable` is everything in between: anything with *some* request or limit set, that isn't a full
`Guaranteed` match.

Now force an OOM kill:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: qosns
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress", "--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    resources:
      requests: {cpu: 50m, memory: 32Mi}
      limits: {cpu: 200m, memory: 64Mi}
EOF

sleep 10
kubectl get pod oom-pod -n qosns
# STATUS: Error or CrashLoopBackOff (it alternates), RESTARTS climbing
```

**Gotcha verified live:** the intuitive move is to check `kubectl describe pod` for a
`Reason: OOMKilled` line — on this cluster, that's not what shows up:
```bash
kubectl describe pod oom-pod -n qosns | grep -A3 "Last State"
# Last State:     Terminated
#   Reason:       Error
#   Exit Code:    137
```
`Reason: Error`, not `Reason: OOMKilled` — the friendlier string isn't guaranteed to appear on
every container runtime/cgroup combination, and this one didn't populate it. **Exit code `137`
(`128 + 9`, i.e. killed by `SIGKILL`) is the reliable signal**, not the `Reason` field's text. A
container that legitimately errors out on its own (an unhandled exception, `exit 1`) shows a
*different*, workload-specific exit code — `137` specifically, on a container whose memory usage
was clearly climbing toward its limit, is what actually points at an OOM kill when the friendlier
label isn't there to lean on.

## Cleanup

```bash
kubectl delete ns qosns
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23. The
`Reason: Error` / exit code `137` result is the same on v1.30.*
