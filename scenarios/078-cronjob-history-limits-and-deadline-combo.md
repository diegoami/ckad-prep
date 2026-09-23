# 078 — CronJob with success/failure history limits and a deadline, all together

**Domain:** Application Design and Build · **Difficulty:** Medium

`026` is a basic CronJob and `062` uses `activeDeadlineSeconds` on its own. This one sets both
history limits plus a deadline, and shows that success and failure history are capped
**independently**, which you only see once you trigger enough runs of both kinds.

## Task

> Create CronJob `batch-report` in namespace `cronhist`, schedule every 2 minutes, keeping only the
> 2 most recent successful Job records and the 2 most recent failed ones — everything older gets
> pruned automatically. Give each run a hard 8 second deadline. Each run starts one container
> `report` (image `busybox:1.31.0`) that runs `sh -c "echo running report; sleep 3"`.

## Documentation

What to look up: **CronJob** — `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#jobs-history-limits> —
  confirms the two limits are independent counters, exactly the behavior this scenario verifies live.

## Setup

```bash
kubectl create ns cronhist
```

## Solution

`successfulJobsHistoryLimit` and `failedJobsHistoryLimit` sit on the CronJob's `spec`, while
`activeDeadlineSeconds` belongs to the Job, so it goes under `jobTemplate.spec`. `backoffLimit: 0`
isn't asked for; it only stops a failing run from being retried, which keeps the demo below quick:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: batch-report
  namespace: cronhist
spec:
  schedule: "*/2 * * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      activeDeadlineSeconds: 8
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: report
            image: busybox:1.31.0
            command: ["sh", "-c", "echo running report; sleep 3"]
          restartPolicy: Never
EOF
```

Now prove the limits work. Trigger three successful runs manually (same `--from=cronjob` trick as
`026`/`062` — a manually created Job still carries an `ownerReference` back to the CronJob, so
history pruning applies to it exactly like a real scheduled run):
```bash
for i in 1 2 3; do
  kubectl create job "batch-report-manual-$i" --from=cronjob/batch-report -n cronhist
  sleep 5
done
sleep 5

kubectl get jobs -n cronhist
# only batch-report-manual-2 and batch-report-manual-3 remain — manual-1 was pruned
# the moment a 3rd successful Job existed, keeping exactly successfulJobsHistoryLimit's worth
```

Now force failures — patch the CronJob's command to a long sleep so `activeDeadlineSeconds: 8` kills
every future run, then trigger three more:
```bash
kubectl patch cronjob batch-report -n cronhist --type=json \
  -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/command","value":["sh","-c","echo run; sleep 300"]}]'
```
**Faster by hand:** `kubectl edit cronjob batch-report -n cronhist`, change the `command:` array
under `spec.jobTemplate.spec.template.spec.containers[0]` directly — much shorter path to eyeball
than the JSON-patch equivalent.
```bash
for i in 1 2 3; do
  kubectl create job "batch-report-fail-$i" --from=cronjob/batch-report -n cronhist
  sleep 12
done

kubectl get jobs -n cronhist
# batch-report-fail-2 and batch-report-fail-3 (both Failed) remain — fail-1 was pruned
# batch-report-manual-2 and -3 (Complete) are STILL there, untouched by the failure-side pruning
# (a scheduled run such as batch-report-29836606 may also appear if the 2-minute mark passed)
```

**Gotcha verified live — the point of this scenario:** the two limits are tracked on completely
separate counters. Creating enough *failed* Jobs to exceed `failedJobsHistoryLimit` never evicts a
*successful* Job, and vice versa — `batch-report-manual-3` (a success) survived every failure added
afterward, because `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` each only prune within
their own outcome bucket. A CronJob can end up holding up to
`successfulJobsHistoryLimit + failedJobsHistoryLimit` Job objects at once, not just
`max()` of the two.

## Cleanup

```bash
kubectl delete ns cronhist
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
