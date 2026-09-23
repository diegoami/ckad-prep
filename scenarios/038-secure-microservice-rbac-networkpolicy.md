# 038 — Harden a microservice: scoped RBAC plus network egress restriction together

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Hard

Combines three concepts (`ServiceAccount`, `Role`/`RoleBinding`, `NetworkPolicy`) that scenarios
006, 023, and 020 each drill in isolation — real hardening work is rarely just one of these.

## Task

> Deployment `payments` (namespace `securesvc`) runs under ServiceAccount `payments-sa` and reads a
> DB password from Secret `db-creds` as an env var. Harden it two ways:
> 1. `payments-sa` should be able to `get`/`list` Secrets in its own namespace (it needs to
>    introspect its own config at runtime) — and nothing else; no Pods, no other verbs.
> 2. Restrict the Pod's egress to only the `database` Service on port 5432, plus DNS.

## Documentation

What to look up: **Using RBAC Authorization** and **Network Policies**, together.
- <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/>

## Setup

```bash
kubectl create ns securesvc
kubectl -n securesvc create serviceaccount payments-sa
kubectl -n securesvc create secret generic db-creds --from-literal=password=s3cret
kubectl -n securesvc create deployment database --image=nginx:1.25-alpine --port=5432
kubectl -n securesvc expose deployment database --port=5432

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: securesvc
spec:
  replicas: 1
  selector: {matchLabels: {app: payments}}
  template:
    metadata: {labels: {app: payments}}
    spec:
      serviceAccountName: payments-sa
      containers:
      - name: payments
        image: nginx:1.25-alpine
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: {name: db-creds, key: password}
EOF
kubectl -n securesvc rollout status deployment payments --timeout=30s
```

kind's default CNI does not enforce NetworkPolicy (see `guide/practice-cluster.md`). Write the
policy correctly regardless of whether you can observe it actually blocking traffic here.

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: securesvc
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-sa-secret-reader
  namespace: securesvc
subjects:
- kind: ServiceAccount
  name: payments-sa
  namespace: securesvc
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

```bash
# confirmed live: exactly the scoped access asked for, nothing broader
kubectl -n securesvc auth can-i get secrets --as=system:serviceaccount:securesvc:payments-sa    # yes
kubectl -n securesvc auth can-i delete secrets --as=system:serviceaccount:securesvc:payments-sa  # no
kubectl -n securesvc auth can-i get pods --as=system:serviceaccount:securesvc:payments-sa        # no
```

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-egress
  namespace: securesvc
spec:
  podSelector:
    matchLabels: {app: payments}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels: {app: database}
    ports:
    - port: 5432
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF
```

Same two-separate-rules shape as scenario 020 and `drills/drill.md` 8.17 — DNS has to be its own
rule, not nested under the `to:` that scopes traffic to `database`. On kind the policy applies
without error but has no effect; the RBAC half is still independently verifiable with
`kubectl auth can-i`, as above.

## Cleanup

```bash
kubectl delete ns securesvc
```
