# 014 — Add a new Secret as env vars, and an existing Secret as a mounted volume, to a running Pod

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

## Task

> The `larch` team runs a Pod named `payment-gateway` in Namespace `larch`. It needs two sets of
> credentials, and both Secrets must live in Namespace `larch` only:
>
> - Create a Secret named `db-credentials` with the keys `username=gateway` and
>   `password=Tr33sAreGr8`. Expose them to the container as the environment variables `DB_USER`
>   and `DB_PASSWORD`.
> - A manifest for a second Secret, `api-keys`, is already in `~/ckad/014/api-keys.yaml`. Create
>   it and mount it into the same container at `/etc/api-keys`, read-only.
>
> Keep the Pod's existing configuration as it is.

## Documentation

What to look up: **Secrets** — consuming as env vars vs. as a mounted volume.
- <https://kubernetes.io/docs/concepts/configuration/secret/> — both the `secretKeyRef` env
  pattern and the volume-mount pattern are covered on this one page.

## Setup

```bash
kubectl create ns larch
mkdir -p ~/ckad/014 && cd ~/ckad/014

# the ready-made manifest for the second Secret
cat > api-keys.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  namespace: larch
stringData:
  payments-key: not-a-real-key-123
  webhook-key: not-a-real-key-456
EOF

# the existing Pod, which already has some configuration of its own
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: payment-gateway
  namespace: larch
  labels: {app: payment-gateway}
spec:
  containers:
  - name: gateway
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1d"]
    env:
    - name: LOG_LEVEL
      value: "info"
EOF
kubectl -n larch wait --for=condition=ready pod/payment-gateway --timeout=60s
```

## Solution

```bash
cd ~/ckad/014
kubectl -n larch create secret generic db-credentials \
  --from-literal=username=gateway --from-literal=password=Tr33sAreGr8
kubectl -n larch apply -f api-keys.yaml
kubectl -n larch get secret

kubectl -n larch get pod payment-gateway -o yaml > payment-gateway.yaml
cp payment-gateway.yaml payment-gateway-new.yaml
```

Edit `payment-gateway-new.yaml`. Strip `status:`, `metadata.uid`/`resourceVersion`/
`creationTimestamp`, `spec.nodeName`, and the auto-mounted `kube-api-access-*` volume and mount
(the same list as in scenario 008). Then add the new env vars and the volume, keeping `LOG_LEVEL`:

```yaml
spec:
  volumes:
  - name: api-keys                    # add
    secret:                           # add
      secretName: api-keys            # add
  containers:
  - name: gateway
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1d"]
    env:
    - name: LOG_LEVEL
      value: "info"
    - name: DB_USER                   # add
      valueFrom:                      # add
        secretKeyRef:                 # add
          name: db-credentials        # add
          key: username               # add
    - name: DB_PASSWORD               # add
      valueFrom:                      # add
        secretKeyRef:                 # add
          name: db-credentials        # add
          key: password               # add
    volumeMounts:
    - name: api-keys                  # add
      mountPath: /etc/api-keys        # add
      readOnly: true                  # add
```

You can't add env vars or volumes to a running Pod in place (`kubectl edit` will reject it and save
a copy to `/tmp`), so replace the Pod:

```bash
kubectl -n larch delete pod payment-gateway --grace-period=0 --force
kubectl -n larch create -f payment-gateway-new.yaml
kubectl -n larch wait --for=condition=ready pod/payment-gateway --timeout=60s

kubectl -n larch exec payment-gateway -- env | grep -E 'DB_|LOG_LEVEL'
# DB_PASSWORD=Tr33sAreGr8
# LOG_LEVEL=info
# DB_USER=gateway      (order may vary)
kubectl -n larch exec payment-gateway -- ls /etc/api-keys
# payments-key
# webhook-key
```

`kubectl replace --force -f payment-gateway-new.yaml` does the delete and create in one step.

## Cleanup

```bash
kubectl delete ns larch
cd ~ && rm -rf ~/ckad/014
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
