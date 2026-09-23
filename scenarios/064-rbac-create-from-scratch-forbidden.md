# 064 — RBAC from scratch: create SA, Role, RoleBinding, and wire them to a Deployment

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Hard

Unlike `048`, where the ServiceAccount and its Role/RoleBinding already exist and only a verb is
wrong, nothing RBAC-related exists here yet. The Pod runs under the `default` ServiceAccount, and
the SA, Role, RoleBinding and the Deployment's `serviceAccountName` all have to be created and
connected from scratch.

## Task

> Namespace `aconcagua` has a Deployment `watcher` whose Pod repeatedly logs a `Forbidden` error
> trying to list Pods in its own namespace. Diagnose why, then fix it: create whatever RBAC objects
> are needed and make sure the Deployment actually uses them, so `watcher`'s logs show a successful
> Pod list instead of a 403.

## Documentation

What to look up: **Using RBAC Authorization**, plus **Configure Service Accounts for Pods**.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/> —
  `spec.serviceAccountName` on a Pod template, the field this scenario's fix ultimately touches.

## Setup

```bash
kubectl create ns aconcagua

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: watcher
  namespace: aconcagua
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
      containers:
      - name: watcher
        image: curlimages/curl:8.10.1
        command: ["sh", "-c"]
        args:
          - |
            while true; do
              TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
              curl -sk -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/aconcagua/pods
              echo
              sleep 30
            done
EOF

kubectl wait --for=condition=Ready pod -l app=watcher -n aconcagua --timeout=60s
```

Confirm the "given" broken state:
```bash
POD=$(kubectl get pods -n aconcagua -l app=watcher -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$POD" -n aconcagua
```
```json
{
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:aconcagua:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"aconcagua\"",
  "reason": "Forbidden",
  "code": 403
}
```
The username in the message — `system:serviceaccount:aconcagua:default` — is the first diagnostic
signal: the Pod is running under the namespace's built-in `default` ServiceAccount, which never has
any permissions granted to it by default. That's the whole diagnosis; everything from here is
building the missing pieces.

## Solution

Create a dedicated ServiceAccount rather than granting the `default` one broad permissions — the
`default` SA is implicitly used by *every* Pod in the namespace that doesn't opt into a different
one, so binding a Role to it would over-grant access to unrelated Pods too:
```bash
kubectl create serviceaccount watcher-sa -n aconcagua
```

Create a Role scoped to exactly what's needed (`get` and `list` on `pods`) and bind it:
```bash
kubectl create role pod-reader -n aconcagua --verb=get,list --resource=pods
kubectl create rolebinding pod-reader-binding -n aconcagua --role=pod-reader --serviceaccount=aconcagua:watcher-sa
```

Confirm the permission itself is correct before touching the Deployment — isolates "is the RBAC
right" from "is the Pod using it":
```bash
kubectl auth can-i list pods --as=system:serviceaccount:aconcagua:watcher-sa -n aconcagua
# yes
```

The step easiest to forget: none of this does anything until the Deployment's Pod template actually
*uses* the new ServiceAccount — `kubectl create serviceaccount` doesn't attach it to anything by
itself:
```bash
kubectl patch deployment watcher -n aconcagua -p \
  '{"spec":{"template":{"spec":{"serviceAccountName":"watcher-sa"}}}}'
```
**Even faster:** `kubectl set serviceaccount deployment/watcher watcher-sa -n aconcagua` is the
dedicated command for exactly this field — see `084` for the full comparison. `kubectl edit
deployment watcher -n aconcagua` (changing `serviceAccountName:` under `spec.template.spec` directly)
works too, if you'd rather stay in one editor session than reach for a separate command.
```bash
kubectl rollout status deployment/watcher -n aconcagua --timeout=30s
```

Re-check the logs on the **new** Pod (the patch triggered a new rollout, so the old Pod's logs will
never change). The old Pod can still be `Terminating` for its 30s grace period, so pick the newest
Pod rather than `{.items[0]}`:
```bash
POD=$(kubectl get pods -n aconcagua -l app=watcher --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')
sleep 5
kubectl logs "$POD" -n aconcagua
# "kind": "PodList" ... — a real list of Pods instead of a Forbidden Status
```

**The trap this scenario is built around:** it's easy to create the SA/Role/RoleBinding trio
correctly, confirm `auth can-i` says `yes`, and declare victory — but `auth can-i --as=` only proves
the *permission* exists for that identity, not that anything in the cluster is actually *using* that
identity. A Pod's `serviceAccountName` is immutable once the Pod is created (Deployments dodge this
by rolling a new Pod on any template change), so this fix always requires a template edit and a
fresh rollout, not just an RBAC-object edit — checking `auth can-i` and stopping there is the single
most common way this task shape goes wrong.

## Cleanup

```bash
kubectl delete ns aconcagua
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
