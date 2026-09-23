# 054 — CrashLoopBackOff: read the crash reason with `--previous`, not plain `logs`

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

`037-multi-bug-triage.md` hits an OOMKilled container as one of several stacked bugs. This scenario
isolates one lesson: why `kubectl logs --previous` exists, and when plain `logs` misleads you.

## Task

> The Pod `worker` in namespace `tagus` keeps restarting. At the moment it shows `READY 1/1`, so it
> looks healthy, but `RESTARTS` keeps going up. Find out why the container died the last time, and
> which exit code it terminated with.

## Documentation

What to look up: **Debug Running Pods** — the `--previous` flag.
- <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/> — reading the
  crashed container's logs, not the new restart attempt's empty ones.

## Setup

```bash
kubectl create ns tagus

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: worker
  namespace: tagus
spec:
  containers:
  - name: worker
    image: busybox
    command: ["sh", "-c"]
    args:
      - |
        echo "attempt started, healthy for now"
        sleep 20
        echo "FATAL: cannot connect to database at db.tagus.svc:5432" >&2
        exit 1
EOF
```

Wait for the first restart before starting the investigation (~20-25s):
```bash
kubectl wait pod/worker -n tagus --for=jsonpath='{.status.containerStatuses[0].restartCount}'=1 --timeout=90s
kubectl get pod worker -n tagus
# NAME     READY   STATUS    RESTARTS     AGE
# worker   1/1     Running   1 (2s ago)   ...
```
(Interactively, `kubectl get pod worker -n tagus -w` and Ctrl-C once `RESTARTS` hits 1 does the
same.)

## Solution

**The trap, verified live:** run plain `kubectl logs` right after that restart, while the *current*
attempt is still in its own healthy startup window (it hasn't reached its own failure point yet) —
it shows nothing useful, because it's showing the **new** attempt's output, not the crash that
already happened:
```bash
kubectl logs worker -n tagus
# attempt started, healthy for now
```
That looks like a healthy, boring pod. It isn't — `RESTARTS` already told you otherwise. The actual
crash reason only lives in the **previous** container instance's log:
```bash
kubectl logs worker -n tagus --previous
# attempt started, healthy for now
# FATAL: cannot connect to database at db.tagus.svc:5432
```

Confirm the Exit Code from `describe` (`Last State`, not `State` — that section is the terminated
instance, this one is live and running):
```bash
kubectl describe pod worker -n tagus | grep -A4 "Last State"
# Last State:  Terminated
#   Reason:    Error
#   Exit Code: 1
```

**Exit Code decoder** (not documented as a single table on `kubernetes.io` — worth memorizing):
| Exit Code | Meaning |
|---|---|
| `0` | Container exited cleanly (success) |
| `1` | Generic application error (the app itself called `exit(1)` or crashed) |
| `137` | `SIGKILL` (128+9) — almost always **OOMKilled**; check `Reason: OOMKilled` in `describe`, see 037 |
| `143` | `SIGTERM` (128+15) — graceful termination requested (pod deletion, `preStop`, or a rolling update); check timing against a deploy/delete, not app logic |

`kubectl logs --previous` only ever holds the **single most-recently-terminated** instance — if
several restarts have happened since the one you care about, that history is already gone (the
container runtime rotates it out), so `--previous` is a narrow, time-sensitive window, not a full
crash history. If you need the full history, `kubectl get events --sort-by='.lastTimestamp'` or
`kubectl describe pod` (which also lists recent Events) is the fallback.

## Cleanup

```bash
kubectl delete ns tagus
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
