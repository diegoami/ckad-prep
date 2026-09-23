# 006 — Retrieve a ServiceAccount's token

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

## Task

> In Namespace `osprey`, the `ledger-sync` ServiceAccount is used by a nightly reconciliation
> service. A colleague debugging that service needs its API token. Find the Secret that holds the
> token for `ledger-sync` and write the token, base64-decoded, to `~/ckad/006/token`.

## Documentation

What to look up: **ServiceAccounts** and how their tokens actually get issued now.
- <https://kubernetes.io/docs/concepts/security/service-accounts/> — the modern, time-bound
  `TokenRequest`-based flow, not the old auto-created long-lived Secret (removed by default in 1.24+).

## Setup

**Version gotcha:** since Kubernetes 1.24, creating a ServiceAccount no longer auto-creates a
long-lived token Secret for it — on a 1.30 cluster a fresh ServiceAccount has zero Secrets. To
reproduce the starting state (a long-lived token Secret already bound to the ServiceAccount),
create the Secret yourself with the `kubernetes.io/service-account.name` annotation; the token
controller fills in `data.token` automatically. A second ServiceAccount with its own token Secret
is there so you have to identify the right one.

```bash
kubectl create ns osprey
kubectl -n osprey create serviceaccount ledger-sync
kubectl -n osprey create serviceaccount ledger-reader

cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: osprey-token-a1
  namespace: osprey
  annotations:
    kubernetes.io/service-account.name: ledger-reader
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: osprey-token-b7
  namespace: osprey
  annotations:
    kubernetes.io/service-account.name: ledger-sync
type: kubernetes.io/service-account-token
YAML

sleep 2   # give the token controller a moment to populate data.token
mkdir -p ~/ckad/006
```

## Solution

The Secret names don't say which ServiceAccount they belong to. The link is the
`kubernetes.io/service-account.name` annotation on the Secret, so list that:

```bash
kubectl -n osprey get secrets
# NAME              TYPE                                  DATA   AGE
# osprey-token-a1   kubernetes.io/service-account-token   3      5s
# osprey-token-b7   kubernetes.io/service-account-token   3      5s

kubectl -n osprey get secrets \
  -o custom-columns='NAME:.metadata.name,SA:.metadata.annotations.kubernetes\.io/service-account\.name'
# NAME              SA
# osprey-token-a1   ledger-reader
# osprey-token-b7   ledger-sync

kubectl -n osprey get secret osprey-token-b7 -o jsonpath='{.data.token}' | base64 -d \
  > ~/ckad/006/token
cat ~/ckad/006/token
# eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

If you can't remember the custom-columns syntax,
`kubectl -n osprey describe secrets | grep -E '^Name:|service-account.name'` gets you the same
pairing. `kubectl describe secret osprey-token-b7` also prints the token **already decoded**. You
could copy it from there, but piping the jsonpath output through `base64 -d` avoids copy-paste
mistakes with a very long string.

Don't expect `kubectl describe sa ledger-sync` to point you at the Secret. Since 1.24 the
ServiceAccount's `secrets` list is no longer populated, so the `Tokens:` line is empty or missing.

If you only need a short-lived token for immediate use rather than a stored Secret, use
`kubectl -n osprey create token ledger-sync`. It's the more common pattern today, even though
this task specifically asks for the long-lived Secret-based one.

## Cleanup

```bash
kubectl delete ns osprey
rm -rf ~/ckad/006
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
