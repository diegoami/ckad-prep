# 057 — Egress and ingress policies both gate a call: fix two clients by relabeling only

**Domain:** Services and Networking · **Difficulty:** Hard

`050` and `070` only involve ingress policies on the server side. Here the namespace also locks
down *egress*, so a call needs permission twice: once from a policy that selects the client, and
once from a policy that selects the server. Two clients each fail on a different side.

## Does your cluster enforce NetworkPolicy?

It depends on your kind version. The kindnet CNI in older kind releases (v0.23 and earlier)
**doesn't enforce NetworkPolicy**: both calls below succeed before you've labeled anything, which
looks like there's nothing to fix. Newer releases enforce it (see `050` for what was observed on
kind v0.31). Run the "given" check in Setup before trusting a result; if nothing gets blocked, see
`guide/practice-cluster.md` for a cluster that enforces it. Reading the policies and choosing the
labels works the same either way.

## Task

> In namespace `ajwain`, two workloads call the `orders-api` Service on port 80: the Pod
> `storefront` and the Pod `invoice-job`. Both calls time out. The NetworkPolicies in `ajwain` are
> owned by the platform team: you may not create, change or delete any of them. Make both calls
> work by changing Pod labels only, and don't remove any label a Pod already has.

## Documentation

What to look up: **Network Policies**, especially how ingress and egress policies combine.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/> — "The two sorts of
  pod isolation": policies are additive, and a connection is allowed only if both the egress policy
  on the source Pod and the ingress policy on the destination Pod allow it.

## Setup

```bash
kubectl create ns ajwain
kubectl run orders-api --image=nginx:1.25-alpine -n ajwain --labels=app=orders-api
kubectl expose pod orders-api -n ajwain --port=80
kubectl run storefront --image=busybox:1.36 -n ajwain \
  --labels=app=storefront,orders-client=true --command -- sleep 3600
kubectl run invoice-job --image=busybox:1.36 -n ajwain \
  --labels=app=invoice-job,orders-access=granted --command -- sleep 3600
kubectl wait --for=condition=Ready pod/orders-api pod/storefront pod/invoice-job -n ajwain --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: baseline-lockdown
  namespace: ajwain
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-egress
  namespace: ajwain
spec:
  podSelector:
    matchLabels:
      orders-access: granted
  policyTypes: ["Egress"]
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: orders-api
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-api-ingress
  namespace: ajwain
spec:
  podSelector:
    matchLabels:
      app: orders-api
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          orders-client: "true"
    ports:
    - protocol: TCP
      port: 80
EOF
```

Confirm the "given" broken state. On a CNI that enforces NetworkPolicy, both calls time out:
```bash
kubectl exec -n ajwain storefront -- wget -qO- -T 3 http://orders-api >/dev/null; echo "exit=$?"
kubectl exec -n ajwain invoice-job -- wget -qO- -T 3 http://orders-api >/dev/null; echo "exit=$?"
# enforcing CNI: "wget: download timed out", exit=1 for both
# kind v0.23 kindnet: exit=0 for both, because nothing is enforced
```

## Solution

List every policy in the namespace, not just the one named after the server:
```bash
kubectl get networkpolicy -n ajwain
# baseline-lockdown    <none>
# orders-api-ingress   app=orders-api
# orders-egress        orders-access=granted
```
Read them as a whole:
- `baseline-lockdown` selects every Pod (`podSelector: {}`) for both directions. Its only rule
  allows egress to port 53, so by itself it blocks all ingress and all egress except DNS.
- `orders-egress` selects Pods labeled `orders-access=granted`. For *those* Pods it adds egress to
  `app=orders-api` on TCP 80.
- `orders-api-ingress` selects `orders-api`. It adds ingress from Pods labeled
  `orders-client=true` on TCP 80.

Policies only ever add allowed traffic, so a call from a client to `orders-api` works when the
client carries **both** labels: `orders-access=granted` (so `orders-egress` selects it and lets the
traffic out) and `orders-client=true` (so `orders-api-ingress` lets it in). Compare with what each
client has:
```bash
kubectl get pods -n ajwain --show-labels
# invoice-job   app=invoice-job,orders-access=granted   <- allowed out, not allowed in
# orders-api    app=orders-api
# storefront    app=storefront,orders-client=true       <- would be allowed in, never gets out
```
Each client is missing the other half:
```bash
kubectl label pod storefront -n ajwain orders-access=granted
kubectl label pod invoice-job -n ajwain orders-client=true
```
`kubectl label` adds labels and leaves the existing ones in place, which the task requires.

Confirm both calls now work:
```bash
kubectl exec -n ajwain storefront -- wget -qO- -T 3 http://orders-api | head -4
kubectl exec -n ajwain invoice-job -- wget -qO- -T 3 http://orders-api | head -4
# <!DOCTYPE html> ... <title>Welcome to nginx!</title>   (both)
```

**The traps:**
- The labels play two different roles. `orders-access=granted` makes the client a *target* of
  `orders-egress` (its `podSelector`), while `orders-client=true` makes it an allowed *source* in
  `orders-api-ingress` (its `from`). Looking only at the server's ingress policy fixes `invoice-job`
  and leaves `storefront` broken, even though `storefront` already "matches" that policy.
- DNS keeps working for both clients throughout. Being selected by `orders-egress` doesn't take away
  the DNS egress that `baseline-lockdown` grants, because egress rules from all policies that select
  a Pod are added together. So a timeout here is a policy problem, not a name-resolution problem.
- The port in the policies is the Pod's port (80 on nginx). The Service port happens to be the same
  here; when it isn't, the policy has to name the `targetPort`.

## Cleanup

```bash
kubectl delete ns ajwain
```

*Re-run on a local kind cluster (Kubernetes v1.30) on 2026-09-24: every command runs cleanly, but
that kindnet does not enforce NetworkPolicy, so the timeouts before relabeling could not be
observed there.*
