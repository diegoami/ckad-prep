# 066 — Rollback trap: plain `rollout undo` can land you back on a *different* broken revision

**Domain:** Application Deployment · **Difficulty:** Medium

`001` has two revisions, so a plain `kubectl rollout undo` lands on the healthy one. Here there are
three (healthy, broken, broken differently), and a plain `undo` only goes back one revision, which
is still broken.

## Task

> Deployment `orders` in namespace `elbrus` (2 replicas) is currently failing. Check its rollout
> history, roll it back to a revision that actually works, and confirm recovery.

## Documentation

What to look up: **Deployments** — `kubectl rollout undo` and `--to-revision`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment> —
  confirms `undo` with no target rolls back to "the previous revision," not to a specific known-good one.

## Setup

On the real exam this broken history already exists. Locally, build it: a healthy revision 1, a
revision 2 broken by a bad container command, then a revision 3 broken a completely different way
by a bad image tag — exactly the "two bad edits in a row" shape that makes plain `undo` insufficient.

```bash
kubectl create ns elbrus

# revision 1 — healthy
kubectl create deployment orders -n elbrus --image=nginx:1.24-alpine --replicas=2
kubectl rollout status deployment orders -n elbrus --timeout=60s

# revision 2 — broken: command overridden to a binary that doesn't exist in the image
kubectl patch deployment orders -n elbrus --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command","value":["/no-such-binary"]}]'
```
**Faster by hand:** `kubectl edit deployment orders -n elbrus`, add `command: ["/no-such-binary"]`
under the container, save.
```bash
# revision 3 — broken a different way: bad image tag, layered on top of revision 2's bad command
kubectl set image deployment/orders nginx=nginx:1.99-does-not-exist -n elbrus
```

Don't wait for that last rollout to finish — it won't.

## Solution

```bash
kubectl rollout status deployment orders -n elbrus --timeout=15s
# error: timed out waiting for the condition

kubectl rollout history deployment orders -n elbrus
# REVISION  CHANGE-CAUSE
# 1
# 2
# 3
```
`rollout history` alone doesn't show *what* differs between revisions or which one is actually
healthy — `--revision=N` inspects one, but the fastest read here is just to try the obvious thing
and verify, not eyeball the diffs.

**The trap:** running `kubectl rollout undo` with no `--to-revision` rolls back to the revision
*immediately before the current one* — that's revision 2, not revision 1, and revision 2 is
*also* broken:
```bash
kubectl rollout undo deployment orders -n elbrus
kubectl rollout status deployment orders -n elbrus --timeout=15s
# error: timed out waiting for the condition — still broken!

kubectl get pods -n elbrus
# new pod stuck in RunContainerError / CrashLoopBackOff — same bad-command bug from revision 2
```
`orders` looks "rolled back" (the command succeeded, the ReplicaSet changed) but the Deployment is
exactly as broken as before, just broken by a different bug. This is the state a candidate under
time pressure is most likely to walk away from thinking the task is done.

Target the actual last-known-good revision explicitly instead:
```bash
kubectl rollout undo deployment orders -n elbrus --to-revision=1
kubectl rollout status deployment orders -n elbrus --timeout=30s
# deployment "orders" successfully rolled out

kubectl get pods -n elbrus
# both pods Running, no RunContainerError / ImagePullBackOff
```

**Lesson:** with more than one bad revision stacked up, a plain `rollout undo` is not a safe
default — always confirm the resulting state with `kubectl rollout status` and `kubectl get pods`
after *any* undo, and if it's still broken, target the specific revision you know was healthy with
`--to-revision` rather than repeatedly running plain `undo` and hoping it walks further back.
Verified live: it doesn't walk backward through history at all — `undo` always targets "whatever's
second-highest in the revision list right now," and each undo consumes the revision it jumps to,
reinserting its content as a *new* top revision. Starting from revisions `1, 2, 3` and undoing once
lands on `2`'s content (history becomes `1, 3, 4`); undoing *again* from there doesn't reach `1` —
it lands on `3`'s content instead (history becomes `1, 4, 5`), which was the *other* broken
revision. Plain `undo`, run repeatedly, oscillates between whatever two revisions are most recent
rather than working its way back to a specific known-good one — `--to-revision` is the only reliable
way to land on revision 1 specifically.

## Cleanup

```bash
kubectl delete ns elbrus
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
