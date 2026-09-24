# 073 — A Service selecting on a label that `kubectl label deployment` never gave the Pods

**Domain:** Services and Networking · **Difficulty:** Medium

`kubectl label deployment X stack=quotes` only labels the **Deployment object's own metadata**. It
never reaches `spec.template.metadata.labels`, so the Pods the Deployment creates don't carry the
label, which matters as soon as a Service is supposed to select on it.

## Task

> The Perseus pricing team runs Deployment `quote-engine` in namespace `perseus` (one container,
> `nginx`, listening on port 80). Make these changes, in this order:
>
> 1. Give the container resource requests of `50m` CPU and `64Mi` memory.
> 2. Label the Deployment `stack=quotes`.
> 3. Scale it to 3 replicas.
>
> Then publish it with a NodePort Service `quote-engine-ext` that listens on port `8080`, forwards
> to the containers' port 80, uses node port `31580`, and has the selector `stack=quotes` and
> nothing else. The Service must end up with 3 endpoints, and a request to node port 31580 must
> return the nginx page.

## Documentation

What to look up: **Labels and Selectors**, plus **Service** (`NodePort`).
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/> — labels live in
  `metadata.labels` on *whatever object* you label; a Deployment and its Pod template are two
  separate `metadata.labels` maps.
- <https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport>

## Setup

```bash
kubectl create ns perseus
kubectl create deployment quote-engine -n perseus --image=nginx:1.25-alpine --replicas=1
kubectl rollout status deployment/quote-engine -n perseus --timeout=30s
```

## Solution

The three Deployment changes each have a dedicated command:
```bash
kubectl set resources deployment quote-engine -n perseus -c nginx --requests=cpu=50m,memory=64Mi
kubectl label deployment quote-engine -n perseus stack=quotes
kubectl scale deployment quote-engine -n perseus --replicas=3
kubectl rollout status deployment/quote-engine -n perseus --timeout=60s
```
`set resources` changes the Pod template, so it starts a rollout; `label` and `scale` don't touch
the template at all. That difference is the whole scenario.

Create the Service. `kubectl expose` can't set a node port, but it can take an explicit selector,
which replaces the one it would otherwise copy from the Deployment (`app=quote-engine`):
```bash
kubectl expose deployment quote-engine -n perseus --name=quote-engine-ext --type=NodePort \
  --port=8080 --target-port=80 --selector=stack=quotes
kubectl get endpoints quote-engine-ext -n perseus
# NAME               ENDPOINTS   AGE
# quote-engine-ext   <none>      0s
```

**The trap this scenario is built around:** no endpoints, because no Pod has `stack=quotes`. Check
where the label actually landed:
```bash
kubectl get deploy quote-engine -n perseus -o jsonpath='metadata.labels: {.metadata.labels}{"\n"}template.labels: {.spec.template.metadata.labels}{"\n"}'
# metadata.labels: {"app":"quote-engine","stack":"quotes"}
# template.labels: {"app":"quote-engine"}          <- stack=quotes never made it here
kubectl get pods -n perseus --show-labels
# quote-engine-db9d67f56-4wttf   1/1   Running   app=quote-engine,pod-template-hash=db9d67f56
# ...                                             <- no stack=quotes on any Pod
```
`kubectl label deployment` only touches the Deployment's own `metadata.labels`. It has nothing to do
with `spec.template.metadata.labels`, the labels stamped onto every Pod. Had you let `expose` copy
the Deployment's selector instead, the Service would have shown 3 endpoints and hidden the problem
until someone compared its selector with the task.

The fix goes at the source: add the label to the Pod **template**. The Deployment rolls out new
Pods that carry it, and the Service picks them up:
```bash
kubectl patch deployment quote-engine -n perseus --type=json \
  -p='[{"op":"add","path":"/spec/template/metadata/labels/stack","value":"quotes"}]'
kubectl rollout status deployment/quote-engine -n perseus --timeout=60s

kubectl get endpoints quote-engine-ext -n perseus
# quote-engine-ext   10.244.0.108:80,10.244.0.109:80,10.244.0.110:80
```
The Deployment's own `spec.selector` (`app=quote-engine`) stays as it is. It's immutable, and it
doesn't need to change: a template may carry more labels than the selector asks for.

Set the node port. `kubectl expose` picked a random one from the 30000–32767 range:
```bash
kubectl patch svc quote-engine-ext -n perseus --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":31580}]'
kubectl get svc quote-engine-ext -n perseus
# NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
# quote-engine-ext   NodePort   10.96.204.149   <none>        8080:31580/TCP
```

Test through the node port, from a throwaway Pod, against the node's IP:
```bash
NODE_IP=$(kubectl get node ckad-control-plane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
kubectl run np-check -n perseus --image=busybox:1.36 --rm -i --restart=Never -- \
  wget -qO- -T 3 "http://$NODE_IP:31580" | grep -o '<title>.*</title>'
# <title>Welcome to nginx!</title>
```

**Gotcha #2 — the target port:** `--target-port=80` matters. Leave it out and `expose` sets
`targetPort` to the same value as `--port` (8080). The endpoints still appear (listed as
`<pod-ip>:8080`), so the Service *looks* healthy, but nothing in the Pods listens on 8080 and every
request fails. When a Service has endpoints but no answers, compare `targetPort` with the
container's real port before anything else. A test from inside the cluster shows it plainly:
`wget: can't connect to remote host (...): Connection refused`.

**Faster by hand for the patches above:** `kubectl edit deployment quote-engine -n perseus` and
`kubectl edit svc quote-engine-ext -n perseus` open each object directly. The label-scope trap has
nothing to do with patch versus edit (it bites just the same in an editor if you add `stack: quotes`
under `metadata.labels` instead of `spec.template.metadata.labels`), but `nodePort` and a template
label are quicker to type in place than as JSON-patch paths.

**Lesson:** `kubectl label <resource>` always targets that resource's own metadata. For a
Deployment, that's a different labels map from the one its Pods inherit. If a label needs to reach
the Pods (for a Service selector, a NetworkPolicy, anything selector-based), it has to go on
`spec.template.metadata.labels`, and `kubectl get pods --show-labels` is the fastest way to confirm
which one you changed.

## Cleanup

```bash
kubectl delete ns perseus
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
