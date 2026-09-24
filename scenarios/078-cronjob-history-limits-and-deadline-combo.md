# 078 — CronJob with a time zone, lopsided success/failure history limits and a per-run deadline

**Domain:** Application Design and Build · **Difficulty:** Medium

`026` is a basic CronJob and `062` uses `activeDeadlineSeconds` on its own. This one sets both
history limits to *different* values plus a deadline, and shows that success and failure history are
capped **independently**, which you only see once you trigger enough runs of both kinds.

## Task

> The `chromium` team needs CronJob `ledger-snapshot` in namespace `chromium`. It runs every 20
> minutes from 06:00 to 22:59, Berlin time. Keep only the single most recent successful Job, but
> keep the last 4 failed ones for troubleshooting. No run may take longer than 25 seconds in total.
> Each run starts one container `snapshot` (image `busybox:1.36`) that runs
> `sh -c "date; echo snapshot written; sleep 2"`.
>
> Then show that the two history limits really are applied separately.

## Documentation

What to look up: **CronJob**: schedule syntax, time zones and `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#jobs-history-limits> —
  confirms the two limits are independent counters, exactly the behavior this scenario verifies live.
- <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#time-zones> — the
  `spec.timeZone` field.

## Setup

```bash
kubectl create ns chromium
```

## Solution

Map each requirement to its field, because they sit at three different levels:
- schedule, time zone and both history limits go on the CronJob's `spec`;
- the 25-second cap is `activeDeadlineSeconds` on the **Job**, so it goes under `jobTemplate.spec`;
- the container goes under `jobTemplate.spec.template.spec`.

`"*/20 6-22 * * *"` fires at :00, :20 and :40 of every hour from 06 to 22, so the last run is
22:40. `backoffLimit: 0` isn't asked for; it only stops a failing run from being retried, which
keeps the demo below quick:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ledger-snapshot
  namespace: chromium
spec:
  schedule: "*/20 6-22 * * *"
  timeZone: "Europe/Berlin"
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 4
  jobTemplate:
    spec:
      activeDeadlineSeconds: 25
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: snapshot
            image: busybox:1.36
            command: ["sh", "-c", "date; echo snapshot written; sleep 2"]
          restartPolicy: Never
EOF

kubectl get cronjob ledger-snapshot -n chromium
# NAME              SCHEDULE          TIMEZONE        SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# ledger-snapshot   */20 6-22 * * *   Europe/Berlin   False     0        <none>          ...
```
`kubectl create cronjob` has no flags for the time zone, the history limits or the deadline, so
either write the YAML directly or generate a skeleton with
`kubectl create cronjob ledger-snapshot --image=busybox:1.36 --schedule="*/20 6-22 * * *" --dry-run=client -o yaml -- sh -c "..."`
and add the rest by hand. Pods have an `activeDeadlineSeconds` field too. Put it in the Pod template
instead of the Job and it only limits each Pod: the Job then starts a replacement Pod (up to
`backoffLimit` times), so the run as a whole isn't capped at 25 seconds.

Now show the limits at work. Trigger two successful runs by hand (a Job created with
`--from=cronjob/...` carries an `ownerReference` back to the CronJob, so history pruning applies to
it just like a scheduled run):
```bash
kubectl create job snap-ok-1 --from=cronjob/ledger-snapshot -n chromium
sleep 8
kubectl create job snap-ok-2 --from=cronjob/ledger-snapshot -n chromium
sleep 10

kubectl get jobs -n chromium
# NAME        STATUS     COMPLETIONS   DURATION   AGE
# snap-ok-2   Complete   1/1           ...
# snap-ok-1 is gone: a second successful Job exceeded successfulJobsHistoryLimit: 1
```

Now make every run fail. Change the command to a long sleep, so the 25-second deadline kills it, then
trigger five more:
```bash
kubectl patch cronjob ledger-snapshot -n chromium --type=json \
  -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/command","value":["sh","-c","echo stuck; sleep 600"]}]'
```
**Faster by hand:** `kubectl edit cronjob ledger-snapshot -n chromium` and change the `command:`
array under `spec.jobTemplate.spec.template.spec.containers[0]`, which is easier to find by eye
than to spell out as a JSON-patch path.
```bash
for i in 1 2 3 4 5; do
  kubectl create job "snap-fail-$i" --from=cronjob/ledger-snapshot -n chromium
  sleep 3
done
sleep 30

kubectl get jobs -n chromium
# snap-fail-2 ... snap-fail-5   Failed   (4 Jobs; snap-fail-1 was pruned)
# snap-ok-2                     Complete (still there)

kubectl describe job snap-fail-5 -n chromium | grep -i deadline
# Active Deadline Seconds:  25s
#   Warning  DeadlineExceeded  ...  job-controller  Job was active longer than specified deadline
```

**Gotcha verified live, and the point of this scenario:** the two limits are tracked separately.
Five failures pruned the oldest *failed* Job and never touched `snap-ok-2`, and the second success
earlier pruned `snap-ok-1` without looking at failures. A CronJob can hold up to
`successfulJobsHistoryLimit + failedJobsHistoryLimit` Jobs at once (here 1 + 4 = 5), not just the
larger of the two.

## Cleanup

```bash
kubectl delete ns chromium
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
