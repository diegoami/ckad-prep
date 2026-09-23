# 009 — Convert a Pod into a Deployment with a container-level SecurityContext

**Domain:** Application Design and Build · **Difficulty:** Medium

## Task

> The `birch` team has been running their `inventory-api` as a bare Pod in Namespace `birch`, and
> it went down during the last node restart. Replace it with a Deployment called `inventory-api`
> running 3 replicas of the same container. As part of a security review, the container in the
> Deployment must run with `allowPrivilegeEscalation: false` and `privileged: false`. Remove the
> original bare Pod once the Deployment is up.

## Documentation

What to look up: **Deployments** — how one wraps a Pod template.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/> — the
  `spec.selector`/`spec.template` shape a bare Pod's spec has to be lifted into.

## Setup

```bash
kubectl create ns birch
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: inventory-api
  namespace: birch
  labels: {app: inventory-api}
spec:
  containers:
  - name: api
    image: nginx:1.27-alpine
    ports:
    - containerPort: 80
EOF
kubectl -n birch wait --for=condition=ready pod/inventory-api --timeout=60s
```

## Solution

```bash
mkdir -p ~/ckad/009 && cd ~/ckad/009
kubectl -n birch get pod inventory-api -o yaml > inventory-api-pod.yaml
cp inventory-api-pod.yaml inventory-api-deployment.yaml
```

Rewrite the copy as a Deployment: move the Pod's `metadata.labels` and `spec` under
`spec.template`, add `replicas` and a `selector` that matches those labels, and add the
securityContext to the container. Drop everything the server generated (`status`, `uid`,
`nodeName`, the `kube-api-access-*` volume and mount, and so on). The result:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-api
  namespace: birch
spec:
  replicas: 3
  selector:
    matchLabels: {app: inventory-api}
  template:
    metadata:
      labels: {app: inventory-api}
    spec:
      containers:
      - name: api
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        securityContext:                   # add
          allowPrivilegeEscalation: false  # add
          privileged: false                # add
```

```bash
kubectl -n birch create -f inventory-api-deployment.yaml
kubectl -n birch rollout status deployment inventory-api --timeout=60s

kubectl -n birch delete pod inventory-api --grace-period=0 --force
kubectl -n birch get pod,deployment
# deployment.apps/inventory-api   3/3   ...
# three inventory-api-<hash>-<id> Pods, no bare inventory-api Pod

kubectl -n birch get deploy inventory-api \
  -o jsonpath='{.spec.template.spec.containers[0].securityContext}{"\n"}'
# {"allowPrivilegeEscalation":false,"privileged":false}
```

The bare Pod has the same `app: inventory-api` label, but the Deployment's ReplicaSet won't adopt
or count it: every ReplicaSet selector also includes a `pod-template-hash` label that the bare Pod
lacks. So you get 3 new Pods alongside the old one, and deleting the old one leaves exactly 3.

## Cleanup

```bash
kubectl delete ns birch
cd ~ && rm -rf ~/ckad/009
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
