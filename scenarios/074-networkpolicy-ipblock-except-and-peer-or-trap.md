# 074 — ipBlock `except`, and a NetworkPolicy `from:` list-vs-single-peer trap that undoes it

**Domain:** Services and Networking · **Difficulty:** Hard

`050`, `057` and `070` only use `podSelector` peers. This one adds an `ipBlock` with `except`, and a
database policy that looks right but lets the whole namespace in, because two peers were written as
separate list entries.

## Task

> A colleague in the `makalu` team drafted two NetworkPolicies and left them in `~/ckad/074/`:
>
> - `database-policy.yaml` should let the `database` Pods (label `app=database`) accept ingress
>   **only** from the `api` Pods (label `app=api`), on TCP port 5432.
> - `frontend-policy.yaml` should let the `frontend` Pods (label `app=frontend`) accept ingress
>   from any IP address except `192.168.1.10/32`.
>
> Apply both in namespace `makalu`, test that each one does what it's meant to, and fix any policy
> that doesn't. Keep the policy names, and leave the fixed versions in the same files.

## Documentation

What to look up: **Network Policies**, specifically `ipBlock` and the OR semantics of the `from`
list.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#targeting-a-namespace-by-its-name> —
  the same `namespaceSelector` field this scenario's bug hinges on.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors> —
  the rule that separate items in one `from`/`to` list are OR'd, not AND'd.

## Setup

```bash
kubectl create ns makalu
kubectl run database --image=busybox:1.31.0 -n makalu --labels=app=database --command -- nc -lk -p 5432 -e true
kubectl run api --image=busybox:1.31.0 -n makalu --labels=app=api --command -- sleep 3600
kubectl run reports --image=busybox:1.31.0 -n makalu --labels=app=reports --command -- sleep 3600
kubectl run frontend --image=nginx:1.25-alpine -n makalu --labels=app=frontend
kubectl expose pod database -n makalu --port=5432
kubectl expose pod frontend -n makalu --port=80
kubectl wait --for=condition=Ready pod/database pod/api pod/reports pod/frontend -n makalu --timeout=60s

# a Pod in a second namespace, to test the namespace boundary
kubectl create ns vinson
kubectl run outsider --image=busybox:1.31.0 -n vinson --command -- sleep 3600
kubectl wait --for=condition=Ready pod/outsider -n vinson --timeout=60s

mkdir -p ~/ckad/074
cat > ~/ckad/074/database-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: makalu
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: makalu
    ports:
    - protocol: TCP
      port: 5432
EOF

cat > ~/ckad/074/frontend-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ip-to-frontend
  namespace: makalu
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 192.168.1.10/32
EOF
```

The blocking described below needs a CNI that enforces NetworkPolicy. kindnet in kind v0.23 and
earlier doesn't, so every connection succeeds there whatever the policies say: you can practise the
steps but not observe the effect. Recent kind releases enforce it (confirmed on kind v0.31). See
`guide/practice-cluster.md`.

## Solution — apply the database policy as drafted, then actually test it

```bash
kubectl apply -f ~/ckad/074/database-policy.yaml

kubectl exec -n makalu api -- nc -zv -w 3 database 5432
# database (...:5432) open   <- correct, api should reach it

kubectl exec -n makalu reports -- nc -zv -w 3 database 5432
# database (...:5432) open   <- WRONG. "reports" is not app=api and still gets through.
```

**The trap:** under `ingress[].from`, `podSelector` and `namespaceSelector` were written as **two
separate list entries** (each with its own leading `-`). Separate entries in a `from` list are
OR'd: a peer only has to match *one* of them. `namespaceSelector: {kubernetes.io/metadata.name:
makalu}` matches every Pod in `makalu`, including `reports`, which has nothing to do with `api`. So
the second entry silently admits the whole namespace and makes the first (`app: api`) pointless.

The namespace boundary itself still holds. A Pod in a different namespace is blocked:
```bash
DBIP=$(kubectl get svc database -n makalu -o jsonpath='{.spec.clusterIP}')
kubectl exec -n vinson outsider -- nc -zv -w 3 "$DBIP" 5432
# no "open" line; exits non-zero after the 3s timeout — correctly blocked
```
So the bug isn't "the policy does nothing". It under-restricts *within* the namespace it's supposed
to narrow down to just `api`.

The fix: drop the `namespaceSelector` entry. A `podSelector` peer without a `namespaceSelector`
already means "Pods in the same namespace as this NetworkPolicy", so the extra entry never added
useful scope, only a backdoor:
```bash
cat > ~/ckad/074/database-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: makalu
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
EOF
kubectl apply -f ~/ckad/074/database-policy.yaml

kubectl exec -n makalu api -- nc -zv -w 3 database 5432
# open — still correct

kubectl exec -n makalu reports -- nc -zv -w 3 database 5432
# no "open" line; times out — now correctly blocked
```
(If the intent had been "`api` Pods AND in a particular namespace" as one combined condition, which
only matters when the `namespaceSelector` targets a *different* namespace from the policy's own,
both keys belong in the *same* `from` entry, not two separate ones. That's a real AND instead of an
OR.)

## The frontend `ipBlock`/`except` policy

```bash
kubectl apply -f ~/ckad/074/frontend-policy.yaml
kubectl exec -n makalu api -- wget -qO- -T 3 http://frontend | head -4
# nginx welcome page — allowed, since the client's (cluster-internal) IP isn't 192.168.1.10
```
This one is correct as drafted. `except` carves a hole out of an otherwise-allowed CIDR:
`0.0.0.0/0` allows any source IP, and the single `/32` exception blocks exactly that one address and
nothing more. There's no practical way to make a Pod's *source* IP equal `192.168.1.10` inside a
kind cluster (Pod IPs come from the cluster's own Pod CIDR), so you can't watch the block fire. The
rule exists to exclude a specific known external client (an office IP, a compromised host) from an
otherwise open ingress rule, which is why it's an `ipBlock` peer rather than a `podSelector` one.

## Cleanup

```bash
kubectl delete ns makalu vinson
rm -rf ~/ckad/074
```

*Verified on kind v0.23 (Kubernetes v1.30) on 2026-09-23, where everything applies cleanly but the
policies aren't enforced, and on kind v0.31 (Kubernetes v1.35), where the blocking was observed as
intended.*
