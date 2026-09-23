# 057 — Two existing NetworkPolicies, three Pods, fix connectivity by labeling only

**Domain:** Services and Networking · **Difficulty:** Hard

`050` has a single NetworkPolicy and a single client Pod to relabel. Here two policies gate a
two-hop chain (`frontend → backend → cache`), so two different Pods each need a different label,
and getting only one right leaves the chain half-broken.

## Does your cluster enforce NetworkPolicy?

It depends on your kind version. The kindnet CNI in older kind releases (v0.23 and earlier)
**doesn't enforce NetworkPolicy**: both hops below work before you've labeled anything, which looks
like there's nothing to fix. Recent releases enforce it: on kind v0.31 (Kubernetes v1.35) both hops
timed out until the labels were added. Run the "given" check in Setup before trusting a result; if
nothing gets blocked, see `guide/practice-cluster.md` for a cluster that enforces it.

## Task

> In namespace `oder`, requests should flow from the Pod `frontend` to the Pod `backend` (Service
> `backend`, port 80) and on from `backend` to the Pod `cache` (Service `cache`, port 80). Both
> hops currently time out. Two NetworkPolicies, `allow-backend-from-frontend` and
> `allow-cache-from-backend`, are in place and must not be modified. Restore the whole chain
> without touching either policy.

## Documentation

What to look up: **Network Policies** — how multiple policies on different Pods compose.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — each policy only
  governs the Pods its own `podSelector` targets; there's no single page section on "chaining," but
  working through the "Behavior of ingress and egress selectors" section makes the per-policy scope clear.

## Setup

```bash
kubectl create ns oder
kubectl run frontend --image=busybox -n oder --labels=app=frontend --command -- sleep 3600
kubectl run backend --image=nginx -n oder --labels=app=backend
kubectl run cache --image=nginx -n oder --labels=app=cache
kubectl expose pod backend -n oder --port=80
kubectl expose pod cache -n oder --port=80
kubectl wait --for=condition=Ready pod/frontend pod/backend pod/cache -n oder --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: oder
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend-client
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cache-from-backend
  namespace: oder
spec:
  podSelector:
    matchLabels:
      app: cache
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend-client
EOF
```

Confirm the "given" broken state — both hops are blocked:
```bash
kubectl exec -n oder frontend -- wget -qO- --timeout=3 http://backend
# wget: download timed out

kubectl exec -n oder backend -- curl -m 3 -s http://cache
# exit code 28 (curl timeout) — empty output
```

## Solution

Each policy's `ingress[].from[].podSelector` names the exact label its *source* traffic needs —
read both policies before labeling anything, don't guess a single label and assume it covers both
hops:
```bash
kubectl get networkpolicy allow-backend-from-frontend -n oder -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels}{"\n"}'
# {"role":"frontend-client"}

kubectl get networkpolicy allow-cache-from-backend -n oder -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels}{"\n"}'
# {"role":"backend-client"}
```

Two separate Pods each need a different label — `backend` needs `role=backend-client` so its
*outbound* traffic to `cache` is allowed (its own `app=backend` label is what makes it a *target*
for the first policy, a separate concern from what label it needs to be a permitted *source* for
the second):
```bash
kubectl label pod frontend role=frontend-client -n oder
kubectl label pod backend role=backend-client -n oder
```

Confirm both hops now work:
```bash
kubectl exec -n oder frontend -- wget -qO- --timeout=3 http://backend | head -3
# <!DOCTYPE html> ... nginx welcome page

kubectl exec -n oder backend -- curl -m 3 -s http://cache | head -3
# <!DOCTYPE html> ... nginx welcome page
```

**The trap this scenario is built around:** labeling only `frontend` fixes the first hop but leaves
`backend → cache` still timing out, because `backend`'s role as a NetworkPolicy *target* (matched by
`app: backend`) and its role as a NetworkPolicy *source* for the next policy (matched by
`role: backend-client`) are governed by two completely independent selectors on two different
policy objects — being an allowed destination for one policy says nothing about being an allowed
source for another. With three Pods and two policies, always check every policy along the path, not
just the one nearest the Pod you're looking at.

## Cleanup

```bash
kubectl delete ns oder
```

*Verified on kind v0.31 (Kubernetes v1.35), where both hops are blocked until the labels are added. Re-run on a local kind cluster (Kubernetes v1.30) on 2026-09-23: every command runs cleanly, but that kindnet does not enforce NetworkPolicy, so the initial block could not be reproduced there.*
