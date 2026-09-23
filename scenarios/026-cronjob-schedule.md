# 026 — Create a CronJob and inspect what it spawns

**Domain:** Application Design and Build · **Difficulty:** Easy

CronJob specifically — the plain `Job` is covered in scenario 004.

## Task

> Create a CronJob `heartbeat` in namespace `cronns` that runs every minute, printing the date,
> using image `busybox:1.31.0`. Confirm it actually spawns a Job and a Pod on schedule, and inspect
> the result.

## Documentation

What to look up: **CronJob**.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/> — cron schedule syntax
  and what a CronJob actually spawns (a Job, which spawns a Pod).

## Setup

```bash
kubectl create ns cronns
```

## Solution

```bash
kubectl -n cronns create cronjob heartbeat --image=busybox:1.31.0 --schedule="*/1 * * * *" -- sh -c "date"

kubectl -n cronns get cronjob heartbeat
# SCHEDULE column confirms it; ACTIVE stays 0 between runs, LAST SCHEDULE updates each minute

# wait past the next minute boundary, then check what it spawned
kubectl -n cronns get cronjob,jobs,pods
```

Within ~60s a `Job` named `heartbeat-<timestamp>` appears with `COMPLETIONS 1/1`,
and its Pod shows `STATUS Completed`. Each scheduled run gets its own Job object — `kubectl -n
cronns get jobs` accumulates one per run (bounded by `spec.successfulJobsHistoryLimit`, default 3).

```bash
# see what one run actually printed
JOB=$(kubectl -n cronns get jobs -o jsonpath='{.items[0].metadata.name}')
kubectl -n cronns logs job/"$JOB"
```

Useful follow-ups worth knowing: `kubectl -n cronns patch cronjob heartbeat -p '{"spec":{"suspend":true}}'`
pauses future scheduling without deleting history; `kubectl -n cronns create job manual-run
--from=cronjob/heartbeat` triggers one run immediately, outside the schedule.

## Cleanup

```bash
kubectl delete ns cronns
```
