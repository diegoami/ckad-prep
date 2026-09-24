# 070 — Let a Deployment through two existing NetworkPolicies by labels alone

**Domain:** Services and Networking · **Difficulty:** Medium

In `057`, two clients are each blocked on a different side (egress or ingress). Here two unrelated
policies each protect a different app, and one client Deployment has to reach both, so its Pods
need what *both* policies ask for at once. The labels have to be read off the policies exactly: one
uses a set-based selector, the other expects a value that isn't `true`, and `kubectl label` accepts
a wrong guess without complaint.

## Task

> In namespace `corvus`, the Corvus finance team runs two internal services, `ledger` (Service
> `ledger`, port 8080) and `ratecard` (Service `ratecard`, port 80). Each is protected by an existing
> NetworkPolicy, `ledger-ingress` and `ratecard-ingress`. The platform team owns those policies:
> they must not be edited, replaced, deleted or supplemented with new ones.
>
> The Deployment `billing-worker` in the same namespace now needs to reach both services. Give its
> Pods access to both, in a way that still holds after the Pods are replaced (a rollout or a
> restart), without touching any NetworkPolicy.

## Documentation

What to look up: **Network Policies**, plus **Labels and Selectors** (set-based requirements).
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — each policy's
  `podSelector`/`ingress[].from` is evaluated independently; a Pod that has to satisfy two separate
  policies needs every label each one asks for.
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement> —
  how `matchExpressions` with `operator: In` reads.

## Setup

```bash
kubectl create ns corvus
kubectl run ledger --image=nginx:1.25-alpine -n corvus --labels=app=ledger
kubectl run ratecard --image=nginx:1.25-alpine -n corvus --labels=app=ratecard
kubectl expose pod ledger -n corvus --port=8080 --target-port=80
kubectl expose pod ratecard -n corvus --port=80
kubectl create deployment billing-worker -n corvus --image=busybox:1.36 --replicas=2 -- sleep 86400
kubectl wait --for=condition=Ready pod/ledger pod/ratecard -n corvus --timeout=60s
kubectl rollout status deployment/billing-worker -n corvus --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ledger-ingress
  namespace: corvus
spec:
  podSelector:
    matchLabels:
      app: ledger
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchExpressions:
        - key: ledger-consumer
          operator: In
          values: ["billing", "audit"]
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ratecard-ingress
  namespace: corvus
spec:
  podSelector:
    matchLabels:
      app: ratecard
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          ratecard-reader: "yes"
EOF
```

Confirm the "given" broken state, with both blocked:
```bash
kubectl exec -n corvus deploy/billing-worker -- wget -qO- -T 3 http://ledger:8080
# wget: download timed out
kubectl exec -n corvus deploy/billing-worker -- wget -qO- -T 3 http://ratecard
# wget: download timed out
```
This needs a CNI that enforces NetworkPolicy. kindnet in kind v0.23 and earlier doesn't, so both
requests succeed even before the fix: you can practise the steps but not observe the block. Recent
kind releases enforce it (confirmed on kind v0.31). See `guide/practice-cluster.md`.

## Solution

Read what each policy wants from a client. `kubectl describe` renders both selector styles
readably:
```bash
kubectl describe networkpolicy ledger-ingress ratecard-ingress -n corvus | grep -A3 'Allowing ingress'
#   Allowing ingress traffic:
#     To Port: 80/TCP
#     From:
#       PodSelector: ledger-consumer in (audit,billing)
# --
#   Allowing ingress traffic:
#     To Port: <any> (traffic allowed to all ports)
#     From:
#       PodSelector: ratecard-reader=yes
```
So a client needs `ledger-consumer` set to `billing` or `audit` (for a billing worker, `billing` is
the honest choice) **and** `ratecard-reader=yes`. Both at the same time, not one or the other.

The labels go on the Deployment's Pod **template**, not on the running Pods. `kubectl label pods -l
app=billing-worker ...` would work right now, but the next rollout or restart creates new Pods from
the template and the access is gone, which the task rules out (see `073` for the same scope issue
with `kubectl label deployment`):
```bash
kubectl patch deployment billing-worker -n corvus \
  -p '{"spec":{"template":{"metadata":{"labels":{"ledger-consumer":"billing","ratecard-reader":"yes"}}}}}'
kubectl rollout status deployment/billing-worker -n corvus --timeout=60s
```
**Faster by hand:** `kubectl edit deployment billing-worker -n corvus` and add the two lines under
`spec.template.metadata.labels`. No trap either way; the strategic-merge patch above just merges
two keys into that map.

Test from one of the new Pods. Select it by the new labels: the old Pods can take a while to
terminate, and they don't carry the labels, so `deploy/billing-worker` could still pick one of
them:
```bash
POD=$(kubectl get pods -n corvus -l ledger-consumer=billing,ratecard-reader=yes -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n corvus "$POD" -- wget -qO- -T 3 http://ledger:8080 | grep -o '<title>.*</title>'
# <title>Welcome to nginx!</title>
kubectl exec -n corvus "$POD" -- wget -qO- -T 3 http://ratecard | grep -o '<title>.*</title>'
# <title>Welcome to nginx!</title>
```

**The trap this scenario is built around:** it's tempting to type a plausible-looking label such as
`ledger-consumer=true` or `ratecard-reader=true` instead of copying what the policy asks for. `kubectl
label` and `kubectl patch` don't check that a label means anything: both report success either way.
Only the `wget` test shows that traffic is still blocked, because `true` is neither `billing`,
`audit` nor `yes`. Always copy keys and values out of the policy's selector, never retype them from
memory or from how the task text phrases it.

**Why `port: 80` in `ledger-ingress` doesn't block the Service port 8080:** a NetworkPolicy sees
traffic after the Service has translated it, so its `ports` refer to the port on the destination
Pod (the Service's `targetPort`, here 80), not to the Service's own port. A client calling
`ledger:8080` arrives at the Pod on port 80 and matches the rule. Changing the policy to 8080
would actually break access.

## Cleanup

```bash
kubectl delete ns corvus
```

*Verified on kind v0.23 (Kubernetes v1.30) on 2026-09-24: Setup, Solution and Cleanup run as
written and the policies apply cleanly, but enforcement wasn't observable because kindnet on kind
v0.23 doesn't enforce NetworkPolicy.*
