# 067 — A manifest with both a removed apiVersion *and* a deprecated field shape

**Domain:** Application Observability and Maintenance · **Difficulty:** Medium

In `031` the field changes come with the new API version. Here the apiVersion bump and the field
reshape are two independent fixes, and doing only one of them still fails.

## Task

> An old HorizontalPodAutoscaler manifest was written against `autoscaling/v2beta2`, which no
> longer exists on this cluster's API server, targeting Deployment `app`. It also uses that API's
> flat `targetAverageUtilization` field for its CPU metric, which `autoscaling/v2` restructured into
> a nested `target` object. Get it applying cleanly on the current stable API.

## Documentation

What to look up: **Deprecated API Migration Guide**, plus **HorizontalPodAutoscaler Walkthrough**.
- <https://kubernetes.io/docs/reference/using-api/deprecation-guide/> — `autoscaling/v2beta1`/
  `v2beta2` removal.
- <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/> — the current
  `autoscaling/v2` metric shape (`target.type`/`target.averageUtilization`).

## Setup

```bash
kubectl create ns dread
kubectl create deployment app -n dread --image=nginx:1.25-alpine --port=80
kubectl rollout status deployment/app -n dread --timeout=30s

cat > old-hpa.yaml <<'EOF'
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: dread
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 60
EOF
```

## Solution

Confirm it's actually rejected — `autoscaling/v2beta1` and `v2beta2` were both removed in
Kubernetes 1.26, well before this cluster's version:
```bash
kubectl apply -f old-hpa.yaml
# error: resource mapping not found for name: "app-hpa" ... no matches for kind
# "HorizontalPodAutoscaler" in version "autoscaling/v2beta2"
```

`kubectl-convert` is a separate plugin that usually isn't installed (see `031`) and, separately,
doesn't handle `autoscaling/*` conversions even when present — this one has to be done by hand
either way, which is worth practising anyway.

**Fixing only the apiVersion is not enough** — bumping the version line alone still fails, and the
error message names the exact field that needs restructuring:
```bash
sed 's/autoscaling\/v2beta2/autoscaling\/v2/' old-hpa.yaml > half-fixed-hpa.yaml
kubectl apply -f half-fixed-hpa.yaml
# Error from server (BadRequest): HorizontalPodAutoscaler in version "v2" cannot be handled
# as a HorizontalPodAutoscaler: strict decoding error: unknown field
# "spec.metrics[0].resource.targetAverageUtilization"
```
`autoscaling/v2` uses strict decoding — unlike some resources where a stale field is silently
dropped, an unrecognized field here is a hard rejection, which is actually a gift: it tells you
precisely which field survived the apiVersion bump but shouldn't have.

Fix both — new apiVersion, and the metric's flat `targetAverageUtilization` nested into
`target: {type: Utilization, averageUtilization: ...}`:
```bash
cat > new-hpa.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: dread
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
EOF

kubectl apply -f new-hpa.yaml
# horizontalpodautoscaler.autoscaling/app-hpa created — proves both fixes were required and correct
```

Confirm:
```bash
kubectl get hpa app-hpa -n dread
# REFERENCE        TARGETS              MINPODS   MAXPODS
# Deployment/app   cpu: <unknown>/60%   1         5
```

The two structural changes `autoscaling/v2beta2` → `v2` always needs on a CPU/memory resource
metric: the version line itself, and every `target<Something>Utilization`/`target<Something>Value`
flat field nested one level deeper under `target: {type: ..., ...}` — the same reshape applies to
every metric entry in the list, not just the first one, if a manifest has more than one.

## Cleanup

```bash
kubectl delete ns dread
rm -f old-hpa.yaml half-fixed-hpa.yaml new-hpa.yaml
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
