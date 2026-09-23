# 063 — Add `runAsUser: 10000` to a Deployment whose securityContext already nests `capabilities`

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

Same core lesson as `051`, but here the existing `securityContext` nests a `capabilities` object
rather than holding two flat booleans. The question is whether a strategic merge patch that adds a
sibling field leaves the nested object intact.

## Task

> Deployment `worker` in namespace `denali` already sets `allowPrivilegeEscalation: false` and
> `capabilities.drop: ["ALL"]` on its one container, `worker`. Add `runAsUser: 10000` without
> removing or changing either existing field.

## Documentation

What to look up: **Configure a Security Context**, plus strategic merge patch behavior.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>
- <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/> —
  how a strategic merge patch handles nested objects, not just flat fields.

## Setup

```bash
kubectl create ns denali

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: denali
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: busybox:1.31.0
        command: ["sh", "-c", "sleep 3600"]
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
EOF

kubectl rollout status deployment/worker -n denali --timeout=60s
```

`busybox` rather than `nginx` here deliberately — `capabilities.drop: ["ALL"]` strips
`NET_BIND_SERVICE` too, and nginx's master process needs it to bind port 80 as root before dropping
privileges; it CrashLoopBackOffs under this exact securityContext. `busybox sleep` needs no
capabilities at all, so the container actually stays up and this stays a pure merge-patch exercise
rather than accidentally also debugging a capability-related crash.

Confirm the "given" state:
```bash
kubectl get deploy worker -n denali -o jsonpath='{.spec.template.spec.containers[0].securityContext}'
# {"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}
```

## Solution

Same rule as `051`: the default strategic merge patch merges nested objects key-by-key, so naming
the container and only the new field is enough:
```bash
kubectl patch deployment worker -n denali -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"worker","securityContext":{"runAsUser":10000}}]}}}}'

kubectl rollout status deployment/worker -n denali --timeout=30s
```

**Faster by hand:** `kubectl edit deployment worker -n denali`, find the existing `securityContext:`
block under the container, add `runAsUser: 10000` as a sibling of `capabilities:`, save — same
result, and (same point as `051`) no merge-type to get wrong since you're never sending a partial patch.

Confirm all three fields survive — `capabilities.drop` included, since it's a nested object one
level deeper than the two fields `051` tested:
```bash
kubectl get deploy worker -n denali -o jsonpath='{.spec.template.spec.containers[0].securityContext}'
# {"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"runAsUser":10000}
```

Confirm it's not just spec-deep but actually enforced at runtime — the container's own process runs
as the new UID:
```bash
POD=$(kubectl get pods -n denali -l app=worker --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')
kubectl exec -n denali "$POD" -- id
# uid=10000 gid=0(root) groups=0(root)
```
Pick the **newest** Pod. `sleep` ignores SIGTERM, so the old Pod stays `Terminating` for its full
30s grace period after `rollout status` returns, and `{.items[0]}` can still pick it. Its `id` shows
`uid=0(root)`, which looks like the patch failed when it hasn't.

The patch's own field ordering — `runAsUser` sitting alongside `capabilities` in the JSON, not
inside it — is what proves the merge is nesting-aware: a patch that clobbered the whole
`securityContext` object instead of merging into it would have silently dropped `capabilities`
entirely, and there'd be no error to catch it. Always re-fetch and check the full field after a
patch touching a nested `securityContext`/`resources`-shaped object — the patch command succeeding
doesn't guarantee sibling fields survived, only checking the result does.

## Cleanup

```bash
kubectl delete ns denali
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
