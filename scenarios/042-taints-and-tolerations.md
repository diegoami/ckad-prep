# 042 — Keep ordinary Pods off a tainted node, admit only Pods that tolerate it

**Domain:** Application Design and Build · **Difficulty:** Medium

The DaemonSet scenario (027) works *around* the built-in control-plane taint with a toleration;
this one uses taints as a deliberate scheduling tool in their own right.

## Task

> Taint the (only) node `dedicated=gpu:NoSchedule`. Confirm an ordinary Pod without a matching
> toleration stays `Pending`, then create one that does tolerate it and confirm it schedules.

## Documentation

What to look up: **Taints and Tolerations**.
- <https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/> — `NoSchedule`/
  `PreferNoSchedule`/`NoExecute` effects and the toleration's `operator: Equal`/`Exists` matching rules.

## Setup

```bash
# assumes the single-node kind cluster from guide/practice-cluster.md
kubectl create ns taintns
```

## Solution

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node "$NODE" dedicated=gpu:NoSchedule
kubectl describe node "$NODE" | grep Taints
# Taints: dedicated=gpu:NoSchedule
```

```bash
kubectl -n taintns run no-toleration --image=nginx:1.25-alpine

kubectl -n taintns get pod no-toleration
# STATUS: Pending
kubectl -n taintns describe pod no-toleration | grep -A2 Events:
# FailedScheduling: 0/1 nodes are available: 1 node(s) had untolerated taint {dedicated: gpu}
```

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
  namespace: taintns
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: app
    image: nginx:1.25-alpine
EOF

kubectl -n taintns get pod with-toleration
# STATUS: Running
```

A toleration doesn't *attract* a Pod to the tainted node — it only *permits* scheduling there
among otherwise-eligible nodes. On a cluster with other, untainted nodes available, a tolerating
Pod could still land anywhere; forcing it specifically onto the tainted node needs a
`nodeSelector`/`nodeAffinity` in addition to the toleration. `operator: Equal` requires `value` to
match exactly; `operator: Exists` (no `value` field) tolerates the key regardless of its value.

## Cleanup

```bash
kubectl delete ns taintns
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node "$NODE" dedicated=gpu:NoSchedule-   # trailing '-' removes the taint
```
