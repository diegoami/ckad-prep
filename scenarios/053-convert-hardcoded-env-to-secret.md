# 053 — Convert a hardcoded env var to a Secret, without touching the other env vars

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

Unlike `014-secret-env-and-volume-on-existing-pod.md`, which adds a *brand-new* Secret-backed env
var, here the env var already exists as a plaintext `value`: you swap it for a `valueFrom` in place
and leave a sibling plain env var untouched.

## Task

> A code review found a database password written in plain text into Deployment `payments` in
> namespace `seine`. Its single container has two env vars: `APP_MODE=production`, which is fine,
> and `DB_PASSWORD`, which holds the password itself. Move the password into a Secret named
> `db-credentials` (key `password`) and make `DB_PASSWORD` read it from there. `APP_MODE` must
> stay exactly as it is.

## Documentation

What to look up: **Secrets** — `valueFrom.secretKeyRef`, plus the patch-merge caveat.
- <https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables>
- <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/> —
  why `value` has to be explicitly nulled before `valueFrom` can be added via patch.

## Setup

```bash
kubectl create ns seine

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: seine
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payments
  template:
    metadata:
      labels:
        app: payments
    spec:
      containers:
      - name: payments
        image: nginx
        env:
        - name: APP_MODE
          value: production
        - name: DB_PASSWORD
          value: hunter2
EOF

kubectl rollout status deployment/payments -n seine --timeout=60s
```

## Solution

Create the Secret first:
```bash
kubectl create secret generic db-credentials -n seine --from-literal=password=hunter2
```

Patch just the `DB_PASSWORD` entry — **explicitly null out `value`** while setting `valueFrom`,
don't just add `valueFrom` alongside it:
```bash
kubectl patch deployment payments -n seine -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"payments","env":[{"name":"DB_PASSWORD","value":null,"valueFrom":{"secretKeyRef":{"name":"db-credentials","key":"password"}}}]}]}}}}'
```

Confirm `APP_MODE` is untouched and `DB_PASSWORD` now resolves from the Secret:
```bash
kubectl rollout status deployment/payments -n seine --timeout=60s
kubectl get deploy payments -n seine -o jsonpath='{.spec.template.spec.containers[0].env}'
# [{"name":"APP_MODE","value":"production"},
#  {"name":"DB_PASSWORD","valueFrom":{"secretKeyRef":{"key":"password","name":"db-credentials"}}}]

# newest Pod: right after the rollout the old one can still be Terminating
POD=$(kubectl get pods -n seine -l app=payments --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1:].metadata.name}')
kubectl exec -n seine "$POD" -- printenv APP_MODE DB_PASSWORD
# production
# hunter2
```

**Gotcha verified live — the trap this scenario is built around:** patching in `valueFrom` without
nulling the existing `value` fails validation outright, it doesn't silently prefer one or the
other:
```
The Deployment "payments" is invalid: spec.template.spec.containers[0].env[1].valueFrom:
Invalid value: "": may not be specified when `value` is not empty
```
An env var may carry `value` *or* `valueFrom`, never both — the strategic merge patch merges the
env entry's fields key-by-key just like any other object, so an old `value` left in place will
collide with a new `valueFrom` unless you explicitly clear it with `"value": null`.

**Faster by hand, same reason as `061`:** `kubectl edit deployment payments -n seine`, find the
`DB_PASSWORD` entry, delete its `value: hunter2` line and replace it with a `valueFrom:` block —
there's no leftover `value` to collide with because you removed it yourself while editing, so the
explicit-null trick above is only something `patch` needs, not something the task inherently requires.

## Cleanup

```bash
kubectl delete ns seine
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
