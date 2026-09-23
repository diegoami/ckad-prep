# 027 — DaemonSet running one Pod per node

**Domain:** Application Design and Build · **Difficulty:** Easy

## Task

> Create a DaemonSet `node-logger` in namespace `dsns`, image `busybox:1.31.0`, that runs exactly
> one Pod on every node in the cluster — including control-plane nodes, which normally repel
> ordinary Pods.

## Documentation

What to look up: **DaemonSet**.
- <https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/> — why one Pod per
  eligible node happens automatically, no `replicas` field involved.

## Setup

```bash
kubectl create ns dsns
```

The kind cluster from `guide/practice-cluster.md` has a single node, so "one per node" here just
means "one, period" — the mechanics are identical on a multi-node cluster, you just don't get to
see the fan-out.

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
  namespace: dsns
spec:
  selector:
    matchLabels: {app: node-logger}
  template:
    metadata:
      labels: {app: node-logger}
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: node-logger
        image: busybox:1.31.0
        command: ["sh", "-c", "while true; do echo alive; sleep 3600; done"]
EOF

kubectl -n dsns get daemonset node-logger
# DESIRED/CURRENT/READY all equal the node count

kubectl -n dsns get pods -o wide
```

Without the `tolerations` entry the DaemonSet's Pod would never schedule onto a control-plane node
(the default `node-role.kubernetes.io/control-plane:NoSchedule` taint repels it, same as any
ordinary Pod) — on a real multi-node cluster that's usually fine (control-plane nodes don't need
the workload), but "every node" in the task literally means every node, taint included.

On a single-node kind cluster you can't prove the toleration is needed: kind removes the
control-plane taint when a cluster has no worker nodes, so the Pod would schedule there even
without it (`kubectl describe node | grep Taints` shows `<none>`). Include it anyway — on the
exam's multi-node clusters the control-plane taint is present.

## Cleanup

```bash
kubectl delete ns dsns
```
