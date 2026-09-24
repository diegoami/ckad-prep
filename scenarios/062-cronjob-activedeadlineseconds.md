# 062 — Put a hard runtime ceiling on every run a CronJob starts

**Domain:** Application Design and Build · **Difficulty:** Medium

Unlike `026` (a CronJob whose Jobs finish cleanly) and `046` (a plain one-off Job), the setting
here has to go into a CronJob's `jobTemplate`, so it governs future runs, not the Job that's
already running.

## Task

> In namespace `nigella`, the CronJob `stock-reconcile` fires every five minutes. Each run waits
> for an upstream marker file before doing its work, and when the upstream system is down the
> marker never appears, so the run just keeps polling. Change the CronJob so that Kubernetes
> terminates any run still active after 25 seconds and records it as failed. Show that it works
> with a run you start by hand.

## Documentation

What to look up: **CronJob**, plus **Jobs**' `activeDeadlineSeconds`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/> — `spec.jobTemplate` is
  where a full Job spec (including `activeDeadlineSeconds`) lives inside a CronJob.
- <https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup>

## Setup

```bash
kubectl create ns nigella

cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: stock-reconcile
  namespace: nigella
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: reconcile
            image: busybox:1.36
            command:
            - sh
            - -c
            - |
              until [ -f /tmp/upstream.done ]; do echo "waiting for upstream"; sleep 5; done
              echo "reconciling stock"
EOF
```

## Solution

Start a run by hand first, before changing anything, to see the problem. `kubectl create job
--from=cronjob/...` starts one immediately instead of waiting for the schedule (same trick as
`026`):
```bash
kubectl create job reconcile-before -n nigella --from=cronjob/stock-reconcile
sleep 40
kubectl get job reconcile-before -n nigella
# STATUS: Running  — well past 25s, and nothing will ever stop it
```

`activeDeadlineSeconds` is a **Job** field, not a CronJob field. On a CronJob it goes one level
down, in `spec.jobTemplate.spec`, so every Job the CronJob creates from now on inherits it:
```bash
kubectl patch cronjob stock-reconcile -n nigella --type=merge \
  -p '{"spec":{"jobTemplate":{"spec":{"activeDeadlineSeconds":25}}}}'
```
**Faster by hand:** `kubectl edit cronjob stock-reconcile -n nigella`, add
`activeDeadlineSeconds: 25` under `spec.jobTemplate.spec` (a sibling of `template:`), save. Putting
it under `spec.jobTemplate.spec.template.spec` instead is a different field: that is the *Pod's*
`activeDeadlineSeconds`, which fails only the Pod, and the Job then starts a replacement.

Start a new run. It has to be a new Job, because the template change doesn't reach Jobs that
already exist:
```bash
kubectl create job reconcile-after -n nigella --from=cronjob/stock-reconcile
kubectl wait --for=condition=Failed job/reconcile-after -n nigella --timeout=60s
# job.batch/reconcile-after condition met

kubectl get job reconcile-after -n nigella -o jsonpath='{.status.conditions[?(@.type=="Failed")].reason}{"\n"}'
# DeadlineExceeded

kubectl get pods -n nigella -l job-name=reconcile-after
# reconcile-after-...   1/1   Terminating   — the Job controller is deleting its Pod
```
The Pod shows `Terminating` for up to 30 seconds: the polling `sh` loop ignores SIGTERM, so it
lives out the default grace period before it is killed. The Job itself is already `Failed`.

**Gotcha verified live:** the run started before the patch is not affected. It is still polling:
```bash
kubectl get job reconcile-before -n nigella
# STATUS: Running

kubectl get job reconcile-before -n nigella -o jsonpath='{.spec.activeDeadlineSeconds}{"\n"}'
# (empty)
```
Each Job gets a copy of the template when it is created, so fixing the CronJob changes future runs
only. That is what "fix it so it doesn't happen again" needs, but don't mistake the old run still
going for the fix not working. If the old run has to go too, delete it:
```bash
kubectl delete job reconcile-before -n nigella
```

## Cleanup

```bash
kubectl delete ns nigella
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
