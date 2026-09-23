# 033 — Attach a debug shell to a running Pod with `kubectl debug`

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

The runnable version of the ephemeral-container mode described in `guide/exam-tips.md`'s
"`kubectl debug` vs `kubectl run`" entry.

## Task

> Pod `target` (namespace `debugns`) is running `nginx:1.25-alpine`, a minimal image with limited
> debugging tools. Without restarting or recreating it, get a shell with more tooling attached to
> it to poke around — as if the app's own image had no shell at all (`kubectl exec` isn't always an
> option).

## Documentation

What to look up: **Ephemeral Containers**, and **Debugging Running Pods**.
- <https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/> — what an ephemeral
  container is/isn't (no probes, no resource guarantees, never restarted).
- <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/> — the
  `kubectl debug --target=` walkthrough, including the "distroless image" use case.

## Setup

```bash
kubectl create ns debugns
kubectl -n debugns run target --image=nginx:1.25-alpine
kubectl -n debugns wait --for=condition=ready pod/target --timeout=30s
```

## Solution

```bash
kubectl -n debugns debug -it target --image=busybox:1.31.0 --target=target -- sh
```

Inside the ephemeral container's shell, `wget`, `nslookup`, etc. from busybox are now available.
Confirmed live, non-interactively:

```bash
kubectl -n debugns debug -it target --image=busybox:1.31.0 --target=target -- \
  sh -c "wget -qO- http://localhost:80 | head -3"
# <!DOCTYPE html>
# <html>
# <head>
```

**Don't credit `--target` for the `localhost:80` reaching nginx above — that's not what it does.**
Every container in a Pod (ephemeral, init, or regular) always shares the Pod's network namespace;
that's just how Pods work, with or without `--target`, with or without `kubectl debug` at all (see
`drills/drill.md` 5.4 for a minimal, ambassador/debug-free example of the same fact). What
`--target=target` actually adds is sharing the *target* container's **process namespace**
specifically — `ps` inside the debug container shows the target's real PIDs, not just its own, and
`/proc/<pid>/root/` becomes a way to browse the target's filesystem. It's a seldom-reached-for
feature, but genuinely the only way to inspect a
running container's processes/filesystem when it has no shell of its own — `kubectl exec` can't get
you there since there's nothing to exec into. Full walkthrough with a real PID list:
`guide/exam-tips.md`'s "`kubectl debug` vs `kubectl run`" entry, not repeated here.

```bash
kubectl -n debugns get pod target -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

The debug container shows up under `spec.ephemeralContainers`, not `spec.containers` — it doesn't
count toward the Pod's restart policy or readiness, and (unlike a regular container) can't be
removed once added; the Pod carries it until the Pod itself is deleted. This is the key difference
from `kubectl run` (creates a brand new standalone Pod) — `kubectl debug` attaches to the Pod
that's already there, in place.

## Cleanup

```bash
kubectl delete ns debugns
```
