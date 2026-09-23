# 020 — Restrict egress from one Deployment to only another, plus DNS

**Domain:** Services and Networking · **Difficulty:** Hard

See also `guide/exam-tips.md` § "NetworkPolicy Egress Gotchas" and `drills/drill.md` 8.17, which
drill the same two-rule shape.

## Does your cluster enforce NetworkPolicy?

It depends on your kind version. The kindnet CNI in older kind releases (v0.23 and earlier) **doesn't
enforce NetworkPolicy**: the policy applies without error, `kubectl describe` shows it as written,
and traffic flows anyway, which looks like success. Recent releases enforce it: kind v0.31
(Kubernetes v1.35, `kindnetd:v20251212-v0.29.0-alpha-105-g20ccfc88`) blocks exactly what this
policy intends. Take the before/after baseline in the Solution rather than assuming either way. If
nothing gets blocked, see `guide/practice-cluster.md` for a cluster that enforces it.

## Task

> In namespace `amber`, the Deployments `inventory-api` and `storefront` each have a Service of
> the same name. For security, `storefront` Pods may only open outgoing connections to
> `inventory-api` Pods. Create a NetworkPolicy named `storefront-egress` that enforces this, but
> still lets `storefront` Pods resolve names through DNS (port 53, UDP and TCP). From a
> `storefront` Pod, `wget inventory-api:9090` should keep working and `wget example.com` should
> fail.

## Documentation

What to look up: **Network Policies** — egress rules, and the DNS-egress gotcha.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — the "Default deny
  all egress traffic" and "SCTP/UDP/TCP" examples section; note the docs explicitly warn that
  restricting egress can silently break DNS unless port 53 is left open.

## Setup

```bash
kubectl create ns amber
kubectl -n amber create deployment inventory-api --image=nginx:1.25-alpine --port=80
kubectl -n amber expose deployment inventory-api --port=9090 --target-port=80

kubectl -n amber create deployment storefront --image=nginx:1.25-alpine --port=80
kubectl -n amber expose deployment storefront --port=80

kubectl -n amber rollout status deployment inventory-api --timeout=60s
kubectl -n amber rollout status deployment storefront --timeout=60s
```

## Solution

```bash
kubectl -n amber get pods --show-labels
# app=inventory-api / app=storefront — those are the labels to select on

STOREFRONT_POD=$(kubectl -n amber get pods -l app=storefront -o jsonpath='{.items[0].metadata.name}')

# baseline: both succeed before the policy exists
kubectl -n amber exec "$STOREFRONT_POD" -- wget -T2 -qO- example.com | head -3
kubectl -n amber exec "$STOREFRONT_POD" -- wget -T2 -qO- inventory-api:9090 | head -3

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: storefront-egress
  namespace: amber
spec:
  podSelector:
    matchLabels:
      app: storefront
  policyTypes:
  - Egress
  egress:
  - to:                      # rule 1: only to inventory-api pods, any port
    - podSelector:
        matchLabels:
          app: inventory-api
  - ports:                   # rule 2: DNS to anywhere — a separate rule, not nested under `to:`
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF

kubectl -n amber describe networkpolicy storefront-egress

# With enforcement: example.com now times out and inventory-api:9090 still works.
# On an old kindnet without enforcement, both still succeed (see the note above).
kubectl -n amber exec "$STOREFRONT_POD" -- wget -T2 -qO- example.com | head -3
kubectl -n amber exec "$STOREFRONT_POD" -- wget -T2 -qO- inventory-api:9090 | head -3
```

Why the rule allows port 80 even though the Service port is 9090: NetworkPolicy works on Pod
traffic after the Service has translated the address. Rule 1 has no `ports:`, so all ports to the
`inventory-api` Pods are allowed. If you added `ports:` to it, you'd have to list the container
port `80`, not the Service port `9090`.

## Cleanup

```bash
kubectl delete ns amber
```

*Verified on kind v0.23 (Kubernetes v1.30) on 2026-09-23, where the policy applies but isn't
enforced, and on kind v0.31 (Kubernetes v1.35), where the blocking was observed as intended.*
