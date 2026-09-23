# 018 — Diagnose and fix a Service with a mismatched selector

**Domain:** Services and Networking · **Difficulty:** Medium

See also `guide/exam-tips.md` § "`kubectl get endpoints` is the fastest way to tell 'Service
misconfigured' from 'DNS/pod broken'".

## Task

> The billing team in namespace `topaz` reports that other workloads can't reach their API. The
> ClusterIP Service `billing-api-svc` should forward port `8081` to the Pods of Deployment
> `billing-api`, but requests to it fail. Reproduce the problem with
> `curl billing-api-svc.topaz:8081` from a temporary Pod, find what's misconfigured and fix it
> so the request succeeds. Don't change the Deployment.

## Documentation

What to look up: **Service** — how `spec.selector` is matched against Pod labels.
- <https://kubernetes.io/docs/concepts/services-networking/service/> — "Defining a Service"; also
  worth knowing `kubectl get endpoints` is the standard first diagnostic, mentioned on the same page.

## Setup

```bash
kubectl create ns topaz
kubectl -n topaz create deployment billing-api --image=nginx:1.25-alpine --port=80
kubectl -n topaz patch deployment billing-api --type=json \
  -p='[{"op":"add","path":"/spec/template/metadata/labels/component","value":"billing-backend"}]'
kubectl -n topaz rollout status deployment billing-api --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: billing-api-svc
  namespace: topaz
spec:
  ports:
  - name: http
    port: 8081
    targetPort: 80
  selector:
    component: billing-api
EOF
```

## Solution

```bash
kubectl -n topaz run tmp --restart=Never --rm -i --image=nginx:alpine -- \
  curl -s -m 5 billing-api-svc.topaz:8081
# fails straight away: connection refused (curl exit code 7) — kube-proxy rejects
# traffic to a Service that has no endpoints

# DNS resolved and the Service exists, so check whether it selected any Pods.
# `get endpoints` is the standard first check.
kubectl -n topaz get endpoints billing-api-svc
# ENDPOINTS: <none>  <- the selector matches nothing

kubectl -n topaz get pods --show-labels
# app=billing-api,component=billing-backend,pod-template-hash=...
kubectl -n topaz describe service billing-api-svc | grep Selector
# Selector: component=billing-api   <- the Pods are labelled component=billing-backend

# fix: point the selector at a label the Pods actually carry
kubectl -n topaz patch service billing-api-svc -p '{"spec":{"selector":{"component":"billing-backend"}}}'

kubectl -n topaz get endpoints billing-api-svc
# now shows <pod-ip>:80

kubectl -n topaz run tmp --restart=Never --rm -i --image=nginx:alpine -- \
  curl -s -m 5 billing-api-svc.topaz:8081
# <!DOCTYPE html> ... Welcome to nginx! ...
```

A strategic-merge patch on `selector` merges keys, it doesn't replace the map. That's fine here
because the key (`component`) stays the same. If the fix meant switching to a different key (for
example to `app: billing-api`), the old wrong key would stay in the selector and still match
nothing. In that case use `kubectl edit`, or `kubectl patch --type=json` with a `replace` on
`/spec/selector`.

## Cleanup

```bash
kubectl delete ns topaz
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
