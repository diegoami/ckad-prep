# 046 — Kill a Job that runs too long, distinct from retrying a Job that fails

**Domain:** Application Design and Build · **Difficulty:** Medium

Scenario 004 tunes `completions`/`parallelism`; `activeDeadlineSeconds` governs *wall-clock time*,
an axis independent of `backoffLimit` (which governs *retry count* on failure).

## Task

> Job `long-job` (namespace `deadlinens`) runs `sleep 300` — but this class of job should never be
> allowed to run longer than 5 seconds in total, regardless of whether it's still making progress.
> Enforce that and confirm it actually gets killed at the deadline, not left running.

## Documentation

What to look up: **Jobs** — `activeDeadlineSeconds` vs. `backoffLimit`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup> —
  the wall-clock deadline vs. retry-count-based termination, covered right next to each other.

## Setup

```bash
kubectl create ns deadlinens
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: long-job
  namespace: deadlinens
spec:
  activeDeadlineSeconds: 5
  backoffLimit: 0
  template:
    spec:
      containers:
      - name: app
        image: busybox:1.31.0
        command: ["sh", "-c", "sleep 300"]
      restartPolicy: Never
EOF

sleep 10
kubectl -n deadlinens get job long-job
# STATUS: Failed — confirmed live, killed at the deadline rather than running the full 300s
kubectl -n deadlinens describe job long-job | grep -A3 Conditions:
# Reason: DeadlineExceeded
```

`activeDeadlineSeconds` is a hard wall-clock ceiling on the Job as a whole (counted from when it
starts, across all retries if any) — it doesn't care whether the container is actively working or
stuck; it kills on a timer either way. `backoffLimit` is unrelated: it caps how many times a
*failed* container gets retried, with no concept of elapsed time. A Job can hit either limit
independently — a fast-failing container hits `backoffLimit` first; a container that's alive but
slow hits `activeDeadlineSeconds` first, exactly as here (`backoffLimit: 0` was set specifically so
only the deadline would be the thing that ends it, not a retry-exhaustion race).

## Cleanup

```bash
kubectl delete ns deadlinens
```
