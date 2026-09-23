# 073 — `kubectl label deployment` labels the object, not the Pods it creates

**Domain:** Services and Networking · **Difficulty:** Medium

`kubectl label deployment X tier=backend` only labels the **Deployment object's own metadata**. It
never reaches `spec.template.metadata.labels`, so the Pods the Deployment creates don't carry the
label, which matters as soon as a Service is supposed to select on it.

## Task

> The `lhotse` team runs Deployment `inventory` (one container, `nginx`). Make these changes:
>
> 1. Label it `tier=backend`.
> 2. Scale it to 4 replicas.
> 3. Add the environment variable `TIER=backend` to the container.
> 4. Change the container image to `alpine:3.20`.
>
> Then expose it with a NodePort Service `inventory-np` on port 80, node port `30073`, whose
> selector uses the `tier=backend` label. The Service must end up with 4 endpoints.

## Documentation

What to look up: **Labels and Selectors**, plus **Service** (`NodePort`).
- <https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/> — labels live in
  `metadata.labels` on *whatever object* you label; a Deployment and its Pod template are two
  separate `metadata.labels` maps.
- <https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport>

## Setup

```bash
kubectl create ns lhotse
kubectl create deployment inventory -n lhotse --image=nginx:1.25-alpine --replicas=1
kubectl rollout status deployment/inventory -n lhotse --timeout=30s
```

## Solution

The label, scale, env and image steps:
```bash
kubectl label deploy inventory tier=backend -n lhotse
kubectl scale deployment inventory -n lhotse --replicas=4
kubectl set env deployment/inventory -n lhotse TIER=backend
kubectl set image deployment/inventory -n lhotse nginx=alpine:3.20
```
Watch the syntax of the last two. `set env` takes each `KEY=value` once, and `set image` takes
`<container>=<image>` pairs only, with nothing else mixed in. A stray extra `KEY=value` argument
is a common copy-paste slip.

**Gotcha #1 — bare `alpine` never finishes rolling out on its own:** the `alpine` image's default
command is a shell with nothing to keep it running, so every new Pod exits straight away and ends
up in `CrashLoopBackOff`, and the Deployment never reaches `4/4`. The task says "change the image",
not "add a command", but a container that exits isn't running the workload, so give it something
to stay up for:
```bash
kubectl patch deployment inventory -n lhotse --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command","value":["sleep","3600"]}]'
kubectl rollout status deployment/inventory -n lhotse --timeout=60s
# deployment "inventory" successfully rolled out
```

**Gotcha #2, and the trap this scenario is built around:** check where `tier=backend` actually
landed:
```bash
kubectl get deploy inventory -n lhotse -o jsonpath='metadata.labels: {.metadata.labels}{"\n"}template.labels: {.spec.template.metadata.labels}{"\n"}'
# metadata.labels: {"app":"inventory","tier":"backend"}
# template.labels: {"app":"inventory"}          <- tier=backend never made it here
```
`kubectl label deployment` only touches the Deployment's own `metadata.labels`. It has nothing to do
with `spec.template.metadata.labels`, the labels stamped onto every Pod. Confirm on the Pods:
```bash
kubectl get pods -n lhotse --show-labels
# app=inventory,pod-template-hash=...           <- no tier=backend on any Pod
```

Expose the Deployment. `kubectl expose` copies the Deployment's **selector** (`app: inventory`)
into the Service, so at this point it works, which can hide the missing label completely:
```bash
kubectl expose deployment inventory -n lhotse --type=NodePort --name=inventory-np --port=80
kubectl get endpoints inventory-np -n lhotse
# 4 endpoints — looks correct!
```

But the task says the selector must use `tier=backend`. Adding it now breaks everything, because
the label still isn't on any Pod:
```bash
kubectl patch svc inventory-np -n lhotse -p '{"spec":{"selector":{"tier":"backend"}}}'
kubectl get endpoints inventory-np -n lhotse
# ENDPOINTS: <none>
```

The real fix goes at the source: add `tier=backend` to the Pod **template**, not the Deployment
object. The Service, which already selects on it, picks the Pods up once they roll:
```bash
kubectl patch deployment inventory -n lhotse --type=json \
  -p='[{"op":"add","path":"/spec/template/metadata/labels/tier","value":"backend"}]'
kubectl rollout status deployment/inventory -n lhotse --timeout=60s

kubectl get endpoints inventory-np -n lhotse
# 4 endpoints again — now matched on tier=backend as well as app=inventory
```

Finally, set the node port:
```bash
kubectl patch svc inventory-np -n lhotse -p '{"spec":{"ports":[{"port":80,"targetPort":80,"nodePort":30073}]}}'
kubectl get svc inventory-np -n lhotse
# PORT(S): 80:30073/TCP
```

**Faster by hand for all the patches above:** `kubectl edit deployment inventory -n lhotse` and
`kubectl edit svc inventory-np -n lhotse` open each object directly. The label-scope trap has
nothing to do with patch versus edit (it bites just the same in an editor if you add `tier:
backend` under `metadata.labels` instead of `spec.template.metadata.labels`), but the mechanical
fields (`command`, `nodePort`, `selector`) are quicker to type in place than as JSON-patch paths.

**Lesson:** `kubectl label <resource>` always targets that resource's own metadata. For a
Deployment, that's a different labels map from the one its Pods inherit. If a label needs to reach
the Pods (for a Service selector, a NetworkPolicy, anything selector-based), it has to go on
`spec.template.metadata.labels`, and `kubectl get pods --show-labels` is the fastest way to confirm
which one you changed.

## Cleanup

```bash
kubectl delete ns lhotse
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
