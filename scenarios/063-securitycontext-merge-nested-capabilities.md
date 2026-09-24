# 063 — Set a container's UID and GID without losing its existing securityContext, in a two-container Pod

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

Same core lesson as `051`, but here the existing `securityContext` nests a `capabilities` object,
and the Pod has a second container that must stay as it is. The question is whether a strategic
merge patch that adds sibling fields leaves both the nested object and the other container intact.

## Task

> The `zedoary` team runs Deployment `invoice-renderer` with two containers, `renderer` and
> `log-shipper`. The `renderer` container already has a `securityContext` with
> `readOnlyRootFilesystem: true` and `capabilities.drop: ["ALL"]`. Make the `renderer` process run
> as user ID `3105` and group ID `3105`. Keep both existing settings, and don't change
> `log-shipper`.

## Documentation

What to look up: **Configure a Security Context**, plus strategic merge patch behavior.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>
- <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/> —
  how a strategic merge patch merges lists of containers by `name` and nested objects key by key.

## Setup

```bash
kubectl create ns zedoary

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: invoice-renderer
  namespace: zedoary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: invoice-renderer
  template:
    metadata:
      labels:
        app: invoice-renderer
    spec:
      containers:
      - name: log-shipper
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
      - name: renderer
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        securityContext:
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
EOF

kubectl rollout status deployment/invoice-renderer -n zedoary --timeout=60s
```

`busybox` rather than `nginx` deliberately: `capabilities.drop: ["ALL"]` also strips
`NET_BIND_SERVICE`, and a read-only root filesystem stops nginx writing its cache and PID files,
so nginx would crash under this securityContext. `sleep` needs neither, which keeps this a pure
merge-patch exercise.

Confirm the "given" state:
```bash
kubectl get deploy invoice-renderer -n zedoary \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}: {.securityContext}{"\n"}{end}'
# log-shipper:
# renderer: {"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":true}
```

## Solution

The default strategic merge patch matches entries in `containers` by `name` and merges nested
objects key by key. Naming the container and only the two new fields is enough. Note that
`renderer` is the *second* container, so `containers/0` in a JSON patch would hit the wrong one:
```bash
kubectl patch deployment invoice-renderer -n zedoary -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"renderer","securityContext":{"runAsUser":3105,"runAsGroup":3105}}]}}}}'

kubectl rollout status deployment/invoice-renderer -n zedoary --timeout=60s
```

**Faster by hand:** `kubectl edit deployment invoice-renderer -n zedoary`, find the
`securityContext:` of the `renderer` container, add `runAsUser: 3105` and `runAsGroup: 3105` next to
`capabilities:`, save. As in `051`, there's no patch type to get wrong because you save the whole
object. Just make sure you are in the right container's block.

Confirm the existing fields survived, the new ones are there, and `log-shipper` is unchanged:
```bash
kubectl get deploy invoice-renderer -n zedoary \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}: {.securityContext}{"\n"}{end}'
# log-shipper:
# renderer: {"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":true,"runAsGroup":3105,"runAsUser":3105}
```

Confirm it is enforced at runtime, not just in the spec:
```bash
POD=$(kubectl get pods -n zedoary -l app=invoice-renderer --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')
kubectl exec -n zedoary "$POD" -c renderer -- id
# uid=3105 gid=3105 groups=3105
kubectl exec -n zedoary "$POD" -c log-shipper -- id
# uid=0(root) gid=0(root) groups=0(root),10(wheel)
```
Pick the **newest** Pod. `sleep` ignores SIGTERM, so the old Pod stays `Terminating` for its full
30s grace period after `rollout status` returns, and `{.items[0]}` can still pick it. Its `id` shows
root, which looks like the patch failed when it hasn't. Also pass `-c`: without it, `kubectl exec`
uses the first container, `log-shipper`.

A patch that *replaced* the whole `securityContext` instead of merging into it would have dropped
`capabilities` and `readOnlyRootFilesystem` without any error. The patch command succeeding doesn't
prove the sibling fields survived; re-reading the object does.

## Cleanup

```bash
kubectl delete ns zedoary
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
