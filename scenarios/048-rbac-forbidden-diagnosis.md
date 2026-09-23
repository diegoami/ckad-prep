# 048 — Diagnose and fix a ServiceAccount hitting `Forbidden`, without recreating anything

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

A common RBAC task shape: the ServiceAccount already has *some* RBAC wired up, but one verb is
missing, and the fix is to patch the existing Role rather than recreate anything. `drills/drill.md`
15.8 drills the same `auth can-i` → inspect → patch loop in isolation.

## Task

> The Pod `watcher` in namespace `danube` runs under the ServiceAccount `metrics-watcher` and polls
> the API server for the Pods in its own namespace. Its logs show the request being rejected with
> `Forbidden`. Fix the permissions so `watcher` can list Pods in `danube`. Don't delete or recreate
> the ServiceAccount, Role, RoleBinding or Pod.

## Documentation

What to look up: **Using RBAC Authorization**, plus `kubectl auth can-i`.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/> — `rules[].verbs`/`resources`.
- <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/> —
  checking a specific identity's permissions without waiting on the Pod itself.

## Setup

```bash
kubectl create ns danube

kubectl create serviceaccount metrics-watcher -n danube

# Deliberately incomplete: only "get", not "list" — this is the bug
kubectl create role pod-reader --verb=get --resource=pods -n danube

kubectl create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=danube:metrics-watcher -n danube

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: watcher
  namespace: danube
spec:
  serviceAccountName: metrics-watcher
  containers:
  - name: watcher
    image: curlimages/curl:8.10.1
    command: ["sh", "-c"]
    args:
      - |
        TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
        curl -sk -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/danube/pods
        sleep 3600
EOF

kubectl wait --for=condition=Ready pod/watcher -n danube --timeout=60s
```

Confirm the "given" broken state — the Pod's own logs contain the exact API server response a
real exam task would describe:
```bash
kubectl logs watcher -n danube
```
```json
{
  "kind": "Status",
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:danube:metrics-watcher\" cannot list resource \"pods\" in API group \"\" in the namespace \"danube\"",
  "reason": "Forbidden",
  "code": 403
}
```

## Solution

Reproduce the denial directly instead of waiting on the Pod again — `auth can-i` gives the same
answer instantly:
```bash
kubectl auth can-i list pods --as=system:serviceaccount:danube:metrics-watcher -n danube
# no
```

Confirm the ServiceAccount *is* bound to a Role (so the fix is the Role's rules, not a missing
RoleBinding):
```bash
kubectl get rolebinding -n danube -o wide
kubectl get role pod-reader -n danube -o yaml
```
`rules[0].verbs` shows only `["get"]` — that's the whole bug.

Patch the existing Role to add the missing verb, in place:
```bash
kubectl patch role pod-reader -n danube --type=json \
  -p='[{"op": "replace", "path": "/rules/0/verbs", "value": ["get", "list"]}]'
```
**Faster by hand:** `kubectl edit role pod-reader -n danube`, add `list` to the existing `verbs:`
array, save — a one-line YAML edit instead of a JSON-patch path expression.

Confirm the fix, both ways:
```bash
kubectl auth can-i list pods --as=system:serviceaccount:danube:metrics-watcher -n danube
# yes

# RBAC changes apply immediately, no Pod restart needed — prove it from inside the SAME Pod:
kubectl exec -n danube watcher -- sh -c \
  'TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token); curl -sk -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/danube/pods' \
  | head -5
# now returns a PodList instead of a Forbidden Status
```

Most Forbidden errors on the exam are a missing verb/resource on an already-existing Role, not a
missing RoleBinding — `kubectl get rolebinding` confirming the binding already exists is what tells
you to look at the Role's `rules` next, instead of reaching for `kubectl create rolebinding` again.

**Where the Pod's identity comes from:** the kubelet mounts a short-lived, automatically rotated
token for the Pod's ServiceAccount at `/var/run/secrets/kubernetes.io/serviceaccount/token`
(`ca.crt` and `namespace` sit next to it). The API server authenticates that token as
`system:serviceaccount:<namespace>:<name>`, and that username is what RBAC evaluates. That's why
`kubectl auth can-i --as=system:serviceaccount:danube:metrics-watcher` gives exactly the answer the
Pod gets. See `006-serviceaccount-token.md` for retrieving a ServiceAccount token yourself.

## Cleanup

```bash
kubectl delete ns danube
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
