# 023 — Restrict a ServiceAccount to read-only Pod access with a Role + RoleBinding

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

## Task

> Team Ceres is building a monitoring tool that needs to read Pods and their logs in namespace
> `ceres`, but nothing else — no editing, no deleting, no access to Secrets or other resources. It
> runs under ServiceAccount `ceres-monitor`. Create a Role scoped to exactly that access, bind it
> to the ServiceAccount, and verify the boundary actually holds (can list pods and read logs,
> cannot delete a pod).

## Documentation

What to look up: **Using RBAC Authorization**.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/> — `Role`/`RoleBinding` (namespace
  -scoped) vs. `ClusterRole`/`ClusterRoleBinding`, and the `rules[].verbs`/`resources` shape.

## Setup

```bash
kubectl create ns ceres
kubectl -n ceres create serviceaccount ceres-monitor
kubectl -n ceres run demo --image=nginx:alpine
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: ceres
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ceres-monitor-pod-reader
  namespace: ceres
subjects:
- kind: ServiceAccount
  name: ceres-monitor
  namespace: ceres
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Verify with `kubectl auth can-i --as=` — this impersonates the ServiceAccount for the check
without needing an actual token, the fastest way to confirm an RBAC boundary on the exam:

```bash
kubectl -n ceres auth can-i list pods --as=system:serviceaccount:ceres:ceres-monitor
# yes
kubectl -n ceres auth can-i get pods/log --as=system:serviceaccount:ceres:ceres-monitor
# yes
kubectl -n ceres auth can-i delete pods --as=system:serviceaccount:ceres:ceres-monitor
# no — confirms the Role is actually scoped, not accidentally broader
```

`--as=system:serviceaccount:<namespace>:<name>` is the impersonation subject format — easy to
mistype (it's not just the SA name). A `Role`/`RoleBinding` pair is namespace-scoped; the
cluster-scoped equivalents are `ClusterRole`/`ClusterRoleBinding`, needed instead if the same
permissions must apply across every namespace.

## Cleanup

```bash
kubectl delete ns ceres
```
