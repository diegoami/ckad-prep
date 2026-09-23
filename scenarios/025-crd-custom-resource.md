# 025 — Discover a CRD and create a Custom Resource instance

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

The exam tests *discovering and using* a CRD that's already installed, not authoring one from
scratch — the Setup installs the CRD only to simulate "an operator already put it there".

## Task

> A CRD for a `Widget` resource (group `ckad.example.com`, short name `wg`) is already installed on
> the cluster by another team's operator. Discover it, then create a `Widget` named `my-widget` in
> namespace `crdtest` with `spec.size: large`.

## Documentation

What to look up: **Custom Resources**.
- <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/> — what a
  CustomResourceDefinition is and how a Custom Resource instance relates to it.

## Setup

```bash
kubectl create ns crdtest
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.ckad.example.com
spec:
  group: ckad.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
    shortNames: ["wg"]
EOF
```

## Solution

```bash
# discover it — the "given" state only says a CRD exists, not its exact name/group
kubectl api-resources | grep -i widget
kubectl explain widget.spec

cat <<'EOF' | kubectl apply -f -
apiVersion: ckad.example.com/v1
kind: Widget
metadata:
  name: my-widget
  namespace: crdtest
spec:
  size: large
EOF

kubectl -n crdtest get widgets
kubectl -n crdtest get wg          # shortName also works, same as any built-in resource
kubectl -n crdtest get widget my-widget -o yaml
```

`kubectl explain` works on a CRD exactly like a built-in resource once it's installed — the schema
defined in `spec.versions[].schema.openAPIV3Schema` is what populates it.

## Cleanup

```bash
kubectl delete ns crdtest
kubectl delete crd widgets.ckad.example.com
```
