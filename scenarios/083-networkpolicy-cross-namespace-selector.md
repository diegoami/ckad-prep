# 083 — Cross-namespace NetworkPolicy via `namespaceSelector`, and the port it actually checks

**Domain:** Services and Networking · **Difficulty:** Hard

In `074` a `namespaceSelector` accidentally matched the policy's *own* namespace. Here it does its
actual job: allowing traffic from a genuinely different namespace, identified by a label on the
Namespace object rather than on any Pod.

## Task

> Namespace `backend-ns` has Pod `api` (label `app=api`) behind Service `api` on port `8080` (target
> port `80`). Namespace `frontend-ns` is labeled `env=frontend`. Create a NetworkPolicy in
> `backend-ns` allowing ingress to `api` only from Pods in namespaces labeled `env=frontend` —
> confirm a Pod in `frontend-ns` can reach it, and a Pod in an unlabeled third namespace can't.

## Documentation

What to look up: **Network Policies** — `namespaceSelector`, and that `ports` matches the
container's port, not the Service's.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#targeting-multiple-namespaces-by-label> —
  cross-namespace matching on a label you put on the Namespace, as here.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#targeting-a-namespace-by-its-name> —
  the built-in `kubernetes.io/metadata.name` label, for when the namespace has no custom label.

## Setup

This scenario needs a CNI that enforces NetworkPolicy. The kindnet CNI in older kind releases
(v0.23 and earlier) **doesn't**: the policy applies without error and all traffic keeps flowing,
so neither the port trap below nor the block on `other-ns` shows up. kind v0.31 enforces it. If
`client-b` can still reach `api` after the policy is fixed, see `guide/practice-cluster.md` for a
cluster that enforces NetworkPolicy.

```bash
kubectl create ns backend-ns
kubectl create ns frontend-ns
kubectl create ns other-ns
kubectl label ns frontend-ns env=frontend

kubectl run api --image=nginx -n backend-ns --labels=app=api
kubectl expose pod api -n backend-ns --port=8080 --target-port=80
kubectl run client-a --image=busybox -n frontend-ns --command -- sleep 3600
kubectl run client-b --image=busybox -n other-ns --command -- sleep 3600
kubectl wait --for=condition=Ready pod/api -n backend-ns --timeout=30s
kubectl wait --for=condition=Ready pod/client-a -n frontend-ns --timeout=30s
kubectl wait --for=condition=Ready pod/client-b -n other-ns --timeout=30s
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-ns
  namespace: backend-ns
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

Test both sides:
```bash
API_IP=$(kubectl get svc api -n backend-ns -o jsonpath='{.spec.clusterIP}')
kubectl exec -n frontend-ns client-a -- wget -qO- --timeout=3 http://$API_IP:8080
# wget: download timed out   <- WRONG, this should have worked
```

**Gotcha verified live — the trap this scenario is built around:** the `namespaceSelector` and the
label were both correct, yet traffic from the *correctly labeled* namespace was still blocked.
The bug is the `ports` entry: `port: 8080` is the **Service's** port, but `NetworkPolicy` filters
traffic at the Pod's network interface, which by the time a packet arrives there has already been
DNAT'd by kube-proxy from the Service's port (`8080`) to the container's actual listening port
(`80`, the Service's `targetPort`). A `NetworkPolicy`'s `ports` list has no concept of Services at
all — it only ever matches the port the *container* is actually listening on:
```bash
kubectl patch networkpolicy allow-from-frontend-ns -n backend-ns --type=json \
  -p='[{"op":"replace","path":"/spec/ingress/0/ports/0/port","value":80}]'
```
**Faster by hand:** `kubectl edit networkpolicy allow-from-frontend-ns -n backend-ns`, change
`port: 8080` to `port: 80` under `ingress[0].ports[0]`, save.
```bash

kubectl exec -n frontend-ns client-a -- wget -qO- --timeout=3 http://$API_IP:8080
# nginx welcome page — now correct

kubectl exec -n other-ns client-b -- wget -qO- --timeout=3 http://$API_IP:8080
# wget: download timed out — correctly still blocked, other-ns has no env=frontend label
```

**Lesson:** whenever a Service's `port` and `targetPort` differ, a `NetworkPolicy`'s `ports` field
must reference `targetPort` (the container's real listening port), never the Service-facing port a
client actually connects to — get this wrong and the policy silently blocks *everyone*, including
traffic that should be allowed, which looks identical to a `podSelector`/`namespaceSelector` mistake
unless you specifically think to check the port.

## Cleanup

```bash
kubectl delete ns backend-ns frontend-ns other-ns
```

*Commands re-run on a local kind cluster (kind v0.23, Kubernetes v1.30) on 2026-09-23. That
cluster doesn't enforce NetworkPolicy, so every request succeeded there; the blocking behaviour
shown above was confirmed on kind v0.31 (Kubernetes v1.35).*
