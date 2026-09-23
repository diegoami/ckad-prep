# 008 — Move a Pod to a different namespace

**Domain:** Core kubectl · **Difficulty:** Medium

## Task

> The online bookshop `leafy-books` is being handed over from the `alder` platform team to the
> `hazel` storefront team. Several web Pods run in Namespace `alder`; only one of them serves
> `leafy-books`. Identify that Pod and relocate it to Namespace `hazel`, keeping its name, labels
> and annotations. Downtime during the move is acceptable. When you're done, the Pod must exist
> only in `hazel`.

## Documentation

What to look up: **Namespaces**, and the fact that a Pod can't be moved in place.
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/> — namespace
  scoping; there's no `kubectl move`, which is the whole reason this scenario is dump/edit/delete/recreate.

## Setup

```bash
kubectl create ns alder
kubectl create ns hazel

kubectl -n alder run web-alder-01 --image=nginx:1.27-alpine \
  --labels="tier=web,id=web-alder-01" \
  --annotations="description=static assets for the internal wiki"
kubectl -n alder run web-alder-02 --image=nginx:1.27-alpine \
  --labels="tier=web,id=web-alder-02" \
  --annotations="description=storefront for the online bookshop leafy-books"
kubectl -n alder run web-alder-03 --image=nginx:1.27-alpine \
  --labels="tier=web,id=web-alder-03" \
  --annotations="description=status page for the payments team"
kubectl -n alder wait --for=condition=ready pod --all --timeout=90s
```

## Solution

```bash
# the Pod names don't give it away; the description annotation does
kubectl -n alder get pod
kubectl -n alder describe pod | grep -E '^Name:|description'
# or in one line:
kubectl -n alder get pod -o custom-columns='NAME:.metadata.name,DESC:.metadata.annotations.description'
# -> web-alder-02 is the leafy-books storefront

mkdir -p ~/ckad/008 && cd ~/ckad/008
kubectl -n alder get pod web-alder-02 -o yaml > web-alder-02.yaml
```

Edit the exported YAML. On a 1.30+ cluster these fields must go before it will recreate cleanly
in another namespace: the whole `status:` block, `metadata.uid`/`resourceVersion`/
`creationTimestamp`, `spec.nodeName`, and the auto-mounted `kube-api-access-<hash>` `projected`
volume under `spec.volumes` plus its matching `volumeMounts` entry in the container. (On older
clusters this used to be a `default-token-xxxxx` Secret volume.) Then change `metadata.namespace`.
The trimmed result looks like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    description: storefront for the online bookshop leafy-books
  labels:
    id: web-alder-02
    tier: web
  name: web-alder-02
  namespace: hazel            # changed from alder
spec:
  containers:
  - image: nginx:1.27-alpine
    name: web-alder-02
    # volumeMounts entry for kube-api-access-* removed
  # volumes: kube-api-access-* entry removed entirely
```

```bash
kubectl -n hazel create -f web-alder-02.yaml
kubectl -n hazel wait --for=condition=ready pod/web-alder-02 --timeout=60s

# remove the original; --grace-period=0 --force skips the graceful shutdown wait,
# a common time-saver in the exam when you don't care about the old Pod
kubectl -n alder delete pod web-alder-02 --grace-period=0 --force

kubectl get pod -A | grep web-alder-02
# hazel   web-alder-02   1/1   Running   ...   (only one line)
```

## Cleanup

```bash
kubectl delete ns alder hazel
cd ~ && rm -rf ~/ckad/008
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
