# 040 — Combine a ConfigMap and a Secret into one mounted directory with a projected volume

**Domain:** Application Design and Build · **Difficulty:** Medium

Scenarios 014/015 mount a ConfigMap and a Secret as *separate* volumes; a `projected` volume
merges multiple sources into a single mount point.

## Task

> Pod `proj-demo` (namespace `projns`) needs both ConfigMap `app-conf` and Secret `app-secret`
> readable as plain files under **one single directory**, `/etc/combined` — not two separate mount
> paths.

## Documentation

What to look up: **Projected Volumes**.
- <https://kubernetes.io/docs/concepts/storage/projected-volumes/> — combining `configMap` and
  `secret` sources under one `projected` volume, and the per-source `items[].path` mapping.

## Setup

```bash
kubectl create ns projns
kubectl -n projns create configmap app-conf --from-literal=env=prod
kubectl -n projns create secret generic app-secret --from-literal=apikey=abc123
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: proj-demo
  namespace: projns
spec:
  containers:
  - name: app
    image: busybox:1.31.0
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: combined
      mountPath: /etc/combined
      readOnly: true
  volumes:
  - name: combined
    projected:
      sources:
      - configMap: {name: app-conf}
      - secret: {name: app-secret}
EOF

kubectl -n projns wait --for=condition=ready pod/proj-demo --timeout=30s
kubectl -n projns exec proj-demo -- ls /etc/combined
# apikey  env  — both sources' keys land as sibling files in the one directory
kubectl -n projns exec proj-demo -- cat /etc/combined/env /etc/combined/apikey
```

A `projected` volume is the only way to get two different resource types (ConfigMap + Secret, or
either combined with a ServiceAccount token / `downwardAPI`) into a single mount without the
application needing to know which file came from which source. Key collisions across sources are
the real danger here — tested live by giving both `app-conf` and `app-secret` a key named `env`:
the Pod comes up `Running` with **no error or warning at all**, and one source's value silently
wins (the later source in `sources:` order, in this test) while the other is dropped entirely. This
is worse than a startup failure, because nothing tells you data went missing — worth deliberately
namespacing key names across sources (`cm-env`, `secret-env`) to avoid ever depending on which one
wins.

## Cleanup

```bash
kubectl delete ns projns
```
