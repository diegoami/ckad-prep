# 084 — RBAC fix without touching RBAC: pick the right ServiceAccount out of several

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

In `048` the existing Role is wrong and gets patched, and in `064` nothing exists and you build the
RBAC from scratch. Here the RBAC objects are off limits: several ServiceAccounts are already bound
to different Roles, and the job is to work out which one grants *exactly* the verb and resource in
the error, then point the workload at it.

## Task

> In namespace `manganese`, the `config-syncer` Deployment keeps logging `Forbidden` errors. The
> platform team owns the namespace's ServiceAccounts, Roles and RoleBindings: you may not create,
> change or delete any of them. Get `config-syncer` working by changing only the Deployment.

## Documentation

What to look up: **Using RBAC Authorization**, plus `kubectl set serviceaccount`.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_serviceaccount/> —
  the dedicated command for swapping which SA a workload uses, no manual patch needed.

## Setup

```bash
kubectl create ns manganese
for sa in syncer pod-inspector config-peek settings-agent; do
  kubectl create serviceaccount "$sa" -n manganese
done
kubectl create role pod-viewer -n manganese --verb=get,list --resource=pods
kubectl create role configmap-getter -n manganese --verb=get --resource=configmaps
kubectl create role configmap-reader -n manganese --verb=get,list,watch --resource=configmaps
kubectl create rolebinding pod-viewer -n manganese --role=pod-viewer --serviceaccount=manganese:pod-inspector
kubectl create rolebinding configmap-getter -n manganese --role=configmap-getter --serviceaccount=manganese:config-peek
kubectl create rolebinding configmap-reader -n manganese --role=configmap-reader --serviceaccount=manganese:settings-agent
kubectl create configmap sync-targets -n manganese --from-literal=targets=eu-west,us-east

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-syncer
  namespace: manganese
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-syncer
  template:
    metadata:
      labels:
        app: config-syncer
    spec:
      serviceAccountName: syncer
      containers:
      - name: config-syncer
        image: curlimages/curl:8.10.1
        command: ["sh", "-c"]
        args:
          - |
            while true; do
              TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
              curl -sk -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/manganese/configmaps
              echo
              sleep 20
            done
EOF

kubectl wait --for=condition=Ready pod -l app=config-syncer -n manganese --timeout=60s
```

## Solution

Start from the log. It names both the identity and the exact permission that's missing:
```bash
POD=$(kubectl get pods -n manganese -l app=config-syncer -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$POD" -n manganese | grep -m1 '"message"'
#   "message": "configmaps is forbidden: User \"system:serviceaccount:manganese:syncer\" cannot list resource \"configmaps\" in API group \"\" in the namespace \"manganese\"",
```
So the Pod runs as `syncer`, and what it needs is **`list` on `configmaps`**. Survey what already
exists. `-o wide` on RoleBindings adds the `ROLE` and `SERVICEACCOUNTS` columns, which gives the
whole picture in one command:
```bash
kubectl get rolebindings -n manganese -o wide
# NAME               ROLE                    AGE   USERS   GROUPS   SERVICEACCOUNTS
# configmap-getter   Role/configmap-getter   ...                    manganese/config-peek
# configmap-reader   Role/configmap-reader   ...                    manganese/settings-agent
# pod-viewer         Role/pod-viewer         ...                    manganese/pod-inspector
```
Two ServiceAccounts look plausible from the names alone. Don't guess from names; ask the API server
about the exact verb and resource from the log, for each candidate:
```bash
for sa in syncer pod-inspector config-peek settings-agent; do
  echo "$sa: $(kubectl auth can-i list configmaps -n manganese --as=system:serviceaccount:manganese:$sa)"
done
# syncer: no
# pod-inspector: no
# config-peek: no
# settings-agent: yes
```
`config-peek` is the decoy: its Role grants `get` on ConfigMaps, which reads one ConfigMap by name
but not the collection, and a `GET .../configmaps` without a name is a `list`. To see everything one
ServiceAccount may do, use
`kubectl auth can-i --list -n manganese --as=system:serviceaccount:manganese:config-peek`.

Swap the ServiceAccount. `kubectl set serviceaccount` is the dedicated command, no need to patch the
Pod template by hand:
```bash
kubectl set serviceaccount deployment/config-syncer settings-agent -n manganese
kubectl rollout status deployment/config-syncer -n manganese --timeout=60s
```

Check the **new** Pod (changing the ServiceAccount changes the Pod template, so there's a new
rollout). The old Pod can still be `Terminating` for up to 30 seconds after `rollout status`
returns, so `items[0]` might pick it; filter on the ServiceAccount instead:
```bash
POD=$(kubectl get pods -n manganese -l app=config-syncer -o jsonpath='{.items[?(@.spec.serviceAccountName=="settings-agent")].metadata.name}')
sleep 5
kubectl logs "$POD" -n manganese | grep -m1 -E '"kind"|"reason"'
#   "kind": "ConfigMapList",   <- success, not a Forbidden Status
```

**Lesson:** not every `Forbidden` is a reason to create RBAC objects. Read the verb and resource out
of the error, list what's already bound (`kubectl get rolebindings -o wide`), and test each candidate
with `kubectl auth can-i` using that exact verb. A Role with the right resource but the wrong verb
(`get` instead of `list`) looks right at a glance and still fails.

## Cleanup

```bash
kubectl delete ns manganese
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
