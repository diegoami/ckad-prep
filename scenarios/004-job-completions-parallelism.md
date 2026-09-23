# 004 — Job with completions, parallelism, and pod labels

**Domain:** Application Design and Build · **Difficulty:** Easy

## Task

> The `kestrel` team wants a batch Job for their nightly reporting. Create it in Namespace
> `kestrel` with the name `nightly-report`. Its single container, named `report-writer`, uses image
> `busybox:1.36` and runs `sleep 3 && echo report written`. The Job must succeed 4 times in total,
> with at most 2 Pods running at once. Every Pod it creates must carry the label
> `pipeline: reporting`. Start the Job, let it finish, and check its events to confirm the Pods
> really ran two at a time.

## Documentation

What to look up: **Jobs** — `completions` and `parallelism` together.
- <https://kubernetes.io/docs/concepts/workloads/controllers/job/> — the "Parallel Jobs" section
  covers exactly this combination, including the different patterns (fixed completion count, work queue).

## Setup

```bash
kubectl create ns kestrel
mkdir -p ~/ckad/004
```

## Solution

```bash
kubectl -n kestrel create job nightly-report --image=busybox:1.36 \
  --dry-run=client -o yaml -- sh -c "sleep 3 && echo report written" > ~/ckad/004/job.yaml
```

`kubectl create job` has no flags for completions, parallelism, pod labels or the container name,
so edit the generated YAML:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nightly-report
  namespace: kestrel
spec:
  completions: 4          # add
  parallelism: 2          # add
  template:
    metadata:
      labels:              # add
        pipeline: reporting  # add
    spec:
      containers:
      - command: ["sh", "-c", "sleep 3 && echo report written"]
        image: busybox:1.36
        name: report-writer   # was: nightly-report
      restartPolicy: Never
```

The label goes under `spec.template.metadata.labels` (the Pod template), not under the Job's own
`metadata.labels`. Only the template's labels end up on the Pods.

```bash
kubectl apply -f ~/ckad/004/job.yaml
kubectl -n kestrel get pods -l pipeline=reporting
# two Pods Running at first, then two more once the first pair completes

kubectl -n kestrel wait --for=condition=complete job/nightly-report --timeout=120s
kubectl -n kestrel get job nightly-report
# NAME             STATUS     COMPLETIONS   DURATION   AGE
# nightly-report   Complete   4/4           13s        13s

kubectl -n kestrel describe job nightly-report
# Parallelism:    2
# Completions:    4
# Pods Statuses:  0 Active (0 Ready) / 4 Succeeded / 0 Failed
# Events: two SuccessfulCreate at the same age, then two more a few seconds later, then Completed

kubectl -n kestrel logs job/nightly-report
# report written
```

## Cleanup

```bash
kubectl delete ns kestrel
rm -rf ~/ckad/004
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
