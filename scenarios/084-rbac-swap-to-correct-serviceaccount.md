# 084 — RBAC fix by swapping to an already-correctly-provisioned ServiceAccount

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

In `048` the existing Role is wrong and gets patched, and in `064` nothing exists and you build the
RBAC from scratch. Here the correct ServiceAccount, Role and RoleBinding already exist, unused; the
only bug is which ServiceAccount the Deployment references.

## Task

> Namespace `corfu` has a Deployment `watcher` whose logs show a `Forbidden` error listing Pods. Two
> ServiceAccounts exist: `app-sa` (currently attached to `watcher`) and `monitor-sa` (already bound
> to a Role granting `get`/`list` on Pods, but not used by anything). Fix `watcher` without creating
> or editing any Role or RoleBinding.

## Documentation

What to look up: **Using RBAC Authorization**, plus `kubectl set serviceaccount`.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_serviceaccount/> —
  the dedicated command for swapping which SA a workload uses, no manual patch needed.

## Setup

```bash
kubectl create ns corfu
kubectl create serviceaccount app-sa -n corfu
kubectl create serviceaccount monitor-sa -n corfu
kubectl create role pod-reader -n corfu --verb=get,list --resource=pods
kubectl create rolebinding pod-reader-binding -n corfu --role=pod-reader --serviceaccount=corfu:monitor-sa

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: watcher
  namespace: corfu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: watcher
  template:
    metadata:
      labels:
        app: watcher
    spec:
      serviceAccountName: app-sa
      containers:
      - name: watcher
        image: curlimages/curl:8.10.1
        command: ["sh", "-c"]
        args:
          - |
            while true; do
              TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
              curl -sk -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/corfu/pods
              echo
              sleep 30
            done
EOF

kubectl wait --for=condition=Ready pod -l app=watcher -n corfu --timeout=60s
```

Confirm the "given" broken state:
```bash
POD=$(kubectl get pods -n corfu -l app=watcher -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$POD" -n corfu | tail -8
```
```json
{
  "message": "pods is forbidden: User \"system:serviceaccount:corfu:app-sa\" cannot list resource
   \"pods\" in API group \"\" in the namespace \"corfu\"",
  "reason": "Forbidden",
  "code": 403
}
```

## Solution

The username in the error — `...serviceaccount:corfu:app-sa` — already tells you which SA is
attached. Before touching anything, survey what RBAC objects actually exist in the namespace:
```bash
kubectl get sa,rolebinding -n corfu
# serviceaccount/app-sa
# serviceaccount/default
# serviceaccount/monitor-sa
# rolebinding.rbac.authorization.k8s.io/pod-reader-binding   Role/pod-reader
```
One RoleBinding, two other ServiceAccounts besides `default` — worth checking whether the
RoleBinding already targets one of them before assuming a Role needs writing:
```bash
kubectl auth can-i list pods --as=system:serviceaccount:corfu:app-sa -n corfu
# no
kubectl auth can-i list pods --as=system:serviceaccount:corfu:monitor-sa -n corfu
# yes
```
`monitor-sa` already has exactly the permission `watcher` needs — the fix is purely which
ServiceAccount the Deployment uses, nothing RBAC-related to create.

`kubectl set serviceaccount` is the dedicated command for this — no need to `kubectl patch` the
Pod template by hand:
```bash
kubectl set serviceaccount deployment/watcher monitor-sa -n corfu
kubectl rollout status deployment/watcher -n corfu --timeout=30s
```

Confirm on the **new** Pod (the SA change triggers a new rollout, same as `064`). The old Pod can
still be `Terminating` for up to 30 seconds after `rollout status` returns, so `items[0]` might
pick the wrong one; filter on the ServiceAccount instead:
```bash
POD=$(kubectl get pods -n corfu -l app=watcher   -o jsonpath='{.items[?(@.spec.serviceAccountName=="monitor-sa")].metadata.name}')
sleep 5
kubectl logs "$POD" -n corfu | grep -m1 '"kind"'
#   "kind": "PodList",   <- success, not a Forbidden Status
```

**Lesson:** not every `Forbidden` log is a signal to start creating RBAC objects — check what
ServiceAccounts and RoleBindings already exist in the namespace first (`kubectl get sa,rolebinding`)
before reaching for `kubectl create role`/`kubectl create rolebinding`. A pre-provisioned,
already-correct SA sitting unused is a common exam shape specifically because it rewards that check
and penalizes skipping straight to "build RBAC from scratch" (`064`'s shape) when it isn't actually
needed here.

## Cleanup

```bash
kubectl delete ns corfu
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
