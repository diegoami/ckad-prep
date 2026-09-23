# 062 — A CronJob whose spawned Jobs must actually exit, not sleep forever

**Domain:** Application Design and Build · **Difficulty:** Medium

Unlike `026` (a CronJob whose Jobs already exit cleanly) and `046` (a plain one-off Job), the bug
here lives inside a CronJob's `jobTemplate`, so the fix has to go into the template that governs
future runs, not into the Job that's already running.

## Task

> CronJob `sweep` in namespace `eiger` runs every minute, but its container never exits on its
> own (`sleep 300` after a debug `echo`) — nothing enforces a runtime limit, so triggered Jobs pile
> up as long-running Pods instead of completing. Fix the CronJob so every future run is killed after
> 8 seconds if it hasn't finished, and confirm a manually-triggered run actually gets killed.

## Documentation

What to look up: **CronJob**, plus **Jobs**' `activeDeadlineSeconds`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/> — `spec.jobTemplate` is
  where a full Job spec (including `activeDeadlineSeconds`) lives inside a CronJob.
- <https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup>

## Setup

```bash
kubectl create ns eiger

cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sweep
  namespace: eiger
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: sweep
            image: busybox:1.31.0
            command: ["sh", "-c", "echo starting sweep; sleep 300"]
          restartPolicy: Never
EOF
```

Confirm the "given" broken state — trigger a run manually rather than waiting on the schedule
(`kubectl create job --from=cronjob/...` fires one immediately, same trick as `026`):
```bash
kubectl create job sweep-manual-1 --from=cronjob/sweep -n eiger
sleep 5
kubectl get jobs,pods -n eiger
# job/sweep-manual-1   Running   0/1   5s   <- still going, nothing will stop it at 300s
kubectl delete job sweep-manual-1 -n eiger
```

## Solution

`activeDeadlineSeconds` belongs on the **Job** spec, not the CronJob spec directly — on a CronJob
it's set one level down, inside `spec.jobTemplate.spec`, so it applies to every Job the CronJob
spawns from here on:
```bash
kubectl patch cronjob sweep -n eiger --type=json \
  -p='[{"op":"add","path":"/spec/jobTemplate/spec/activeDeadlineSeconds","value":8}]'
```
**Faster by hand:** `kubectl edit cronjob sweep -n eiger`, add `activeDeadlineSeconds: 8` under
`spec.jobTemplate.spec`, save — the JSON-patch path above is just locating the same field.

Trigger a fresh manual run — it has to be a *new* Job, since patching the CronJob's template doesn't
retroactively affect a Job that's already running:
```bash
kubectl create job sweep-manual-2 --from=cronjob/sweep -n eiger
sleep 20
kubectl get job sweep-manual-2 -n eiger
# STATUS: Failed — killed well before the 300s sleep would have finished

kubectl describe job sweep-manual-2 -n eiger | grep -A2 Events -A5
# Warning  DeadlineExceeded  ...  Job was active longer than specified deadline
```

**Gotcha verified live:** any run *already in flight* when you patch the CronJob keeps going
unaffected — the real scheduled ticks (fired by the `*/1 * * * *` schedule while the
manual testing was happening) were still `Running` well past 8 seconds, because they'd started
*before* the patch landed and only the `jobTemplate` for *future* Job creations changed. If the task
says "fix it so it doesn't happen again," patching the CronJob is correct and sufficient — but don't
expect an already-running Pod to react to it, and don't mistake that for the fix not having worked.

## Cleanup

```bash
kubectl delete ns eiger
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
