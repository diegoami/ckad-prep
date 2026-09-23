# 061 — Service selector using the wrong label *key*, not just the wrong value

**Domain:** Services and Networking · **Difficulty:** Medium

In `018` the selector has the right key (`component`) with the wrong value. Here the Service uses a
key (`run`) the Pods don't have at all: the classic result of copying a selector from a
`kubectl run` Pod (auto-labelled `run=<name>`) onto Pods created by `kubectl create deployment`
(auto-labelled `app=<name>`).

## Task

> In namespace `shannon`, the Deployment `web` was created with `kubectl create deployment`, so its
> Pods are labelled `app=web`. The Service `web` (port 80) in front of it doesn't route any traffic:
> requests to it are refused. Find what's wrong with the Service and fix it so
> `wget http://web` from inside the namespace returns the nginx page.

## Documentation

What to look up: **Service** — selectors, plus the default labels `kubectl run`/`kubectl create
deployment` apply.
- <https://kubernetes.io/docs/concepts/services-networking/service/> — `spec.selector` must match
  Pod labels by *key and value*; the docs' own examples consistently use `app` as the convention,
  which is why `run=` (from `kubectl run`'s older default) is an easy mismatch to introduce.

## Setup

```bash
kubectl create ns shannon
kubectl create deployment web -n shannon --image=nginx:1.25-alpine --port=80
kubectl rollout status deployment/web -n shannon --timeout=60s

# Deliberately broken: selector key copied from a `kubectl run` Pod's default label,
# but this Deployment's Pods were never labeled `run=web` — only `app=web`
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: shannon
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    run: web
EOF
```

Confirm the "given" broken state:
```bash
kubectl get endpoints web -n shannon
# ENDPOINTS: <none>

kubectl run tmp --restart=Never --rm -i --image=busybox -n shannon -- wget -qO- --timeout=3 http://web
# wget: can't connect to remote host ... Connection refused
```

## Solution

`kubectl get endpoints` (or `kubectl get ep`, the short name) is the standard first check for "the
Service exists but nothing's behind it" — confirm the mismatch, then compare label keys, not just
values:
```bash
kubectl get pods -n shannon --show-labels
# app=web,pod-template-hash=...          <- no "run" label anywhere

kubectl describe service web -n shannon | grep Selector
# Selector:  run=web
```

**Gotcha verified live — a naive strategic-merge patch doesn't fix this the way it fixed `018`:**
patching in the correct key without explicitly clearing the old one *adds* a second required key
instead of replacing the selector:
```bash
kubectl patch service web -n shannon -p '{"spec":{"selector":{"app":"web"}}}'
kubectl describe service web -n shannon | grep Selector
# Selector:  app=web,run=web    <- both keys now required, still matches nothing
```
Because `spec.selector` is a plain map (not a list with a merge key), a strategic merge patch adds
or overwrites individual map *keys* — it never removes one you didn't mention. Since the label
*key* itself is changing (`run` → `app`), not just its value, the old `run: web` entry has to be
explicitly nulled out in the same patch:
```bash
kubectl patch service web -n shannon -p '{"spec":{"selector":{"run":null,"app":"web"}}}'
```

Confirm:
```bash
kubectl describe service web -n shannon | grep Selector
# Selector:  app=web

kubectl get endpoints web -n shannon
# ENDPOINTS: 10.244.x.x:80

kubectl run tmp --restart=Never --rm -i --image=busybox -n shannon -- wget -qO- --timeout=3 http://web
# nginx welcome page
```

**Faster by hand, and it avoids the null-key trick entirely:** `kubectl edit service web -n shannon`
shows the existing `selector: {run: web}` right there in your editor — just delete the `run: web`
line and type `app: web` in its place, save. You're replacing the whole map as you see it, not
computing a patch that has to explicitly null out a key you can't currently see; the "which key do I
need to null" reasoning above only exists because `patch` sends a diff against state you're not looking at.

The same `"key": null` trick used in `053` to clear a conflicting `value` before setting
`valueFrom` applies here too — any time a strategic merge patch needs to *remove* a map key rather
than add or change one, it has to be nulled explicitly, whether that map is `env[].value` or
`spec.selector`.

## Cleanup

```bash
kubectl delete ns shannon
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
