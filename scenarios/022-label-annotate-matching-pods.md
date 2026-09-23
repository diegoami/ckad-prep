# 022 — Label and annotate Pods based on an existing label match

**Domain:** Core kubectl · **Difficulty:** Easy

## Task

> Before a cleanup job runs in namespace `quartz`, some Pods need to be marked as off-limits. Add
> the label `retain: "true"` to every Pod in `quartz` whose `role` label is `batch` or `ingest`.
> Then add the annotation `retain-reason: "needed for quarterly audit"` to every Pod that now
> carries `retain: "true"`. No other Pod should change.

## Documentation

What to look up: **Labels and Selectors**.
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/> — label syntax and
  the `-l`/`--selector` flag `kubectl label`/`kubectl annotate` both accept to target a matching set.

## Setup

```bash
kubectl create ns quartz
kubectl -n quartz run nightly-batch --image=nginx:1.25-alpine --labels="role=batch"
kubectl -n quartz run event-ingest --image=nginx:1.25-alpine --labels="role=ingest"
kubectl -n quartz run web-frontend --image=nginx:1.25-alpine --labels="role=web"
kubectl -n quartz wait --for=condition=ready pod --all --timeout=60s
```

## Solution

```bash
kubectl -n quartz get pod --show-labels

# one set-based selector covers both values
kubectl -n quartz label pod -l 'role in (batch,ingest)' retain=true
# (two equality selectors work just as well: -l role=batch, then -l role=ingest)

kubectl -n quartz annotate pod -l retain=true retain-reason="needed for quarterly audit"

kubectl -n quartz get pod -L role,retain
# NAME            ROLE     RETAIN     (other columns trimmed)
# event-ingest    ingest   true
# nightly-batch   batch    true
# web-frontend    web

kubectl -n quartz get pod -l retain=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.metadata.annotations.retain-reason}{"\n"}{end}'
# event-ingest  needed for quarterly audit
# nightly-batch  needed for quarterly audit
```

`web-frontend` (`role=web`) should have neither the label nor the annotation. Check it explicitly:

```bash
kubectl -n quartz get pod web-frontend -o jsonpath='{.metadata.labels}{"\n"}{.metadata.annotations}{"\n"}'
# {"role":"web"}
# (empty line: no annotations)
```

## Cleanup

```bash
kubectl delete ns quartz
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
