# 070 — One new Pod needs two different labels to satisfy two independent NetworkPolicies

**Domain:** Services and Networking · **Difficulty:** Medium

In `057`, two policies gate a chain and two different Pods each need one label. Here two unrelated
policies each protect a different app, and a single client Pod has to reach both, so it needs both
labels at once. The label keys have to be copied exactly: a key that is off by one hyphen still
"works" with `kubectl label` and still gets blocked.

## Task

> The `everest` namespace runs two apps, `catalog` and `checkout`, each behind a Service on port
> 80 and each protected by an existing NetworkPolicy (`catalog-access` and `checkout-ingress`).
> Those policies are owned by the security team and must not be edited, replaced or supplemented.
>
> A new Pod `reporting` in the same namespace needs to reach both the `catalog` and the `checkout`
> Service. Give it access to both without creating, changing or deleting any NetworkPolicy.

## Documentation

What to look up: **Network Policies**.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — each policy's
  `podSelector`/`ingress[].from` is evaluated independently; a Pod that has to satisfy two separate
  policies needs every label each one asks for.

## Setup

```bash
kubectl create ns everest
kubectl run catalog --image=nginx:1.25-alpine -n everest --labels=app=catalog
kubectl run checkout --image=nginx:1.25-alpine -n everest --labels=app=checkout
kubectl run reporting --image=busybox:1.31.0 -n everest --labels=app=reporting --command -- sleep 3600
kubectl expose pod catalog -n everest --port=80
kubectl expose pod checkout -n everest --port=80
kubectl wait --for=condition=Ready pod/catalog pod/checkout pod/reporting -n everest --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: catalog-access
  namespace: everest
spec:
  podSelector:
    matchLabels:
      app: catalog
  ingress:
  - from:
    - podSelector:
        matchLabels:
          catalogaccess: "true"
  egress:
  - to:
    - podSelector:
        matchLabels:
          catalogaccess: "true"
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: checkout-ingress
  namespace: everest
spec:
  podSelector:
    matchLabels:
      app: checkout
  ingress:
  - from:
    - podSelector:
        matchLabels:
          checkout-client: "true"
  policyTypes:
  - Ingress
EOF
```

Confirm the "given" broken state, with both blocked:
```bash
kubectl exec -n everest reporting -- wget -qO- -T 3 http://catalog
# wget: download timed out
kubectl exec -n everest reporting -- wget -qO- -T 3 http://checkout
# wget: download timed out
```
This needs a CNI that enforces NetworkPolicy. kindnet in kind v0.23 and earlier doesn't, so both
requests succeed even before the fix: you can practise the steps but not observe the block. Recent
kind releases enforce it (confirmed on kind v0.31). See `guide/practice-cluster.md`.

## Solution

Read each policy's `ingress[].from[].podSelector`. They use two *different* label keys, and
`reporting` needs both at the same time, not one or the other:
```bash
kubectl get networkpolicy catalog-access -n everest -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels}{"\n"}'
# {"catalogaccess":"true"}

kubectl get networkpolicy checkout-ingress -n everest -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels}{"\n"}'
# {"checkout-client":"true"}
```

One `kubectl label` call, both keys at once:
```bash
kubectl label pod reporting catalogaccess=true checkout-client=true -n everest
```

Confirm both paths now work:
```bash
kubectl exec -n everest reporting -- wget -qO- -T 3 http://catalog | head -4
# nginx welcome page

kubectl exec -n everest reporting -- wget -qO- -T 3 http://checkout | head -4
# nginx welcome page
```

**The trap this scenario is built around:** it's tempting to type a plausible-looking label such as
`catalog-access=true` (hyphenated, like the policy's own name) instead of copying the policy's
selector key verbatim. `kubectl label` doesn't check that a key means anything: `pod/reporting
labeled` reports success either way. Only the `wget` test shows that traffic is still blocked,
because `catalog-access` and `catalogaccess` are two different strings that look almost the same.
Always copy the exact key out of `podSelector.matchLabels`, never retype it from memory or from how
the task text phrases it.

`catalog-access` also has an `egress` rule (restricted to `catalogaccess: "true"` peers). It governs
the `catalog` Pod's own *outbound* connections, not who may connect to it. It doesn't matter here,
because `reporting` only opens connections *to* `catalog`, and replies to an allowed connection are
always allowed. Worth noticing so you don't spend time on a rule that doesn't apply to the direction
of traffic being asked about.

## Cleanup

```bash
kubectl delete ns everest
```

*Verified on kind v0.23 (Kubernetes v1.30) on 2026-09-23, where everything applies cleanly but the
policies aren't enforced, and on kind v0.31 (Kubernetes v1.35), where the blocking was observed as
intended.*
