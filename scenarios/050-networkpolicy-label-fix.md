# 050 — Restore connectivity blocked by an existing NetworkPolicy, by relabeling the client Pod

**Domain:** Services and Networking · **Difficulty:** Easy

An existing NetworkPolicy, which the task says not to edit, only admits traffic from Pods with a
certain label. The fix is entirely on the client Pod's labels, never the policy.

## Does your cluster enforce NetworkPolicy?

It depends on your kind version. The kindnet CNI in older kind releases (v0.23 and earlier)
**doesn't enforce NetworkPolicy**: the policy applies without error and traffic flows anyway, so
the "broken" state below never appears. Recent releases enforce it: on kind v0.31 (Kubernetes
v1.35, `kindnetd:v20251212-v0.29.0-alpha-105-g20ccfc88`) the `wget` from the unlabeled Pod timed
out with the policy applied and succeeded once the client was labeled correctly (and, separately,
once the policy was deleted). Run the "given" check in Setup before trusting a result. If nothing
gets blocked, see `guide/practice-cluster.md` for a cluster that enforces it; the Task and Solution
still work as authoring practice.

## Task

> In namespace `volga`, the Pod `client` (no labels) must reach the Pod `db` through the Service
> `db` on port 80, but its requests time out. A NetworkPolicy `db-allow-frontend` controls ingress
> to `db`. You're not allowed to change the policy. Make `client` able to reach `db`.

## Documentation

What to look up: **Network Policies** — `ingress[].from[].podSelector`.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — how a `from`
  peer's `podSelector` matches on the *client's* labels, not the target's.

## Setup

```bash
kubectl create ns volga
kubectl run db --image=nginx --labels=app=db -n volga
kubectl expose pod db -n volga --port=80
kubectl run client --image=busybox -n volga --command -- sleep 3600
kubectl wait --for=condition=Ready pod/db pod/client -n volga --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-frontend
  namespace: volga
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
EOF
```

Confirm the "given" broken state:
```bash
kubectl exec -n volga client -- wget -qO- --timeout=3 http://db
# wget: download timed out
# (on a kindnet that doesn't enforce NetworkPolicy you get the nginx page instead — see above)
```

## Solution

Read the policy's `from` selector — it names the exact label the client is missing, don't guess:
```bash
kubectl get networkpolicy db-allow-frontend -n volga -o yaml
# spec.ingress[0].from[0].podSelector.matchLabels: role: frontend
```

Label the client Pod (not the policy):
```bash
kubectl label pod client role=frontend -n volga
```

Confirm:
```bash
kubectl exec -n volga client -- wget -qO- --timeout=3 http://db | head -3
# <!DOCTYPE html> ... nginx welcome page
```

This is more common on the exam than writing a NetworkPolicy from scratch: the policy is already
correct and off-limits, and the fix is only ever a missing/wrong label on the client Pod.

## Cleanup

```bash
kubectl delete ns volga
```

*Verified on kind v0.31 (Kubernetes v1.35), where the policy blocks and the label fixes it. Re-run on a local kind cluster (Kubernetes v1.30) on 2026-09-23: every command runs cleanly, but that kindnet does not enforce NetworkPolicy, so the initial block could not be reproduced there.*
