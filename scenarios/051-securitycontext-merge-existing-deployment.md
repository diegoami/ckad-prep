# 051 — Add `runAsUser` to an existing Deployment without losing its other securityContext fields

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

There is no `kubectl set securityContext`. The only tools are `kubectl patch` and `kubectl edit`,
and it's easy to replace the whole `securityContext` block (or the whole `containers` list) by
accident instead of adding one field to it.

## Task

> The security team wants the `api` container of Deployment `api` in namespace `elbe` to run as UID
> `1000`. The container already has a securityContext with `allowPrivilegeEscalation: false` and
> `readOnlyRootFilesystem: false`. Add `runAsUser: 1000` to it and keep both existing settings
> exactly as they are.

## Documentation

What to look up: **Configure a Security Context**, plus `kubectl patch`'s merge behavior.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/> — `runAsUser` field placement.
- <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/> —
  strategic merge patch vs. JSON merge patch, the exact distinction this scenario's gotcha hinges on.

## Setup

```bash
kubectl create ns elbe

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: elbe
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
EOF

kubectl rollout status deployment/api -n elbe --timeout=60s
```

Confirm the "given" state:
```bash
kubectl get deploy api -n elbe -o jsonpath='{.spec.template.spec.containers[0].securityContext}'
# {"allowPrivilegeEscalation":false,"readOnlyRootFilesystem":false}
```

## Solution

Default `kubectl patch` (strategic merge patch) merges nested objects key-by-key rather than
replacing them, so naming the container and only the new field is enough — the two existing
fields survive:
```bash
kubectl patch deployment api -n elbe -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"api","securityContext":{"runAsUser":1000}}]}}}}'
```

Confirm all three fields are present:
```bash
kubectl get deploy api -n elbe -o jsonpath='{.spec.template.spec.containers[0].securityContext}'
# {"allowPrivilegeEscalation":false,"readOnlyRootFilesystem":false,"runAsUser":1000}
```

**Gotcha verified live — the trap this scenario is built around:** re-running the *same* patch
content with `--type=merge` (plain JSON Merge Patch, RFC 7386) instead of the default strategic
merge fails outright:
```bash
kubectl patch deployment api -n elbe --type=merge -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"api","securityContext":{"runAsUser":2000}}]}}}}'
# The Deployment "api" is invalid: spec.template.spec.containers[0].image: Required value
```
`--type=merge` doesn't understand the strategic-merge list-merge-key convention — it treats
`containers` as a single list to replace wholesale, so a container object with only `name` and
`securityContext` and no `image` is invalid, and the API server rejects the whole patch (the
Deployment is left unchanged, not silently corrupted — but the patch simply doesn't work). Leave
`--type` at its default for anything touching a `containers[]` list; reach for `--type=json`
(JSON Patch, targeting an explicit `/spec/.../containers/0/securityContext/runAsUser` path) instead
if you need an unambiguous single-field change.

**Faster by hand, and it sidesteps this entire gotcha:** `kubectl edit deployment api -n elbe`
opens the *whole* object in your terminal editor — you're editing and saving back the complete spec,
not sending a partial patch, so there's no merge-type (`strategic` vs. `merge` vs. `json`) to pick
correctly in the first place. Add `runAsUser: 1000` under the existing `securityContext:` block,
save, done — the strategic-vs-merge trap above simply doesn't exist when you're not patching. This
file uses `patch` because it needs to run non-interactively for verification; on the actual exam,
`edit` is usually the faster and safer choice for exactly this kind of "add one field to an existing
nested block" change.

## Cleanup

```bash
kubectl delete ns elbe
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
