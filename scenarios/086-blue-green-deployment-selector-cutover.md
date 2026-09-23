# 086 — Blue/green deployment: instant full cutover via a Service selector patch

**Domain:** Application Deployment · **Difficulty:** Medium

The curriculum names blue/green and canary side by side. Canary (`028`, `060`, `072`) is a
gradual, ratio-based overlap; blue/green is an **instant, all-or-nothing** cutover with no overlap,
built on the same Service-selector mechanism.

## Task

> Namespace `bluegreen` has Deployment `app-blue` (label `version: blue`) live behind Service
> `app-svc`. Deploy a new version as Deployment `app-green` (label `version: green`), verify it's
> healthy *before* it receives any real traffic, then cut `app-svc` over to it completely in one
> atomic step — no gradual ramp, no overlap.

## Documentation

What to look up: **Service** (selectors), and **Managing Resources**' deployment-strategy section.
- <https://kubernetes.io/docs/concepts/services-networking/service/> — a Service's `spec.selector`
  is the whole mechanism blue/green cutover relies on; there's no dedicated "blue/green" page.
- <https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments> —
  the closest official page, for contrast with canary's replica-ratio approach.

## Setup

```bash
kubectl create ns bluegreen

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: bluegreen
spec:
  replicas: 3
  selector:
    matchLabels: {app: myapp, version: blue}
  template:
    metadata:
      labels: {app: myapp, version: blue}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=blue-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: bluegreen
spec:
  selector: {app: myapp, version: blue}
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl rollout status deployment/app-blue -n bluegreen --timeout=60s
```

## Solution

Deploy green as a **completely separate** Deployment — same `app` label so it *could* be selected,
different `version` label so it currently *isn't*:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: bluegreen
spec:
  replicas: 3
  selector:
    matchLabels: {app: myapp, version: green}
  template:
    metadata:
      labels: {app: myapp, version: green}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=green-response", "-listen=:5678"]
        ports:
        - containerPort: 5678
EOF

kubectl rollout status deployment/app-green -n bluegreen --timeout=60s
```

Verify green is healthy **independently of the Service** — hit a green Pod's IP directly, not
through `app-svc`, since `app-svc` shouldn't be routing to it yet:
```bash
GREEN_IP=$(kubectl get pods -n bluegreen -l version=green -o jsonpath='{.items[0].status.podIP}')
kubectl run tmp --restart=Never --rm -i --image=busybox -n bluegreen -- wget -qO- --timeout=3 http://$GREEN_IP:5678
# green-response

# confirm app-svc is still 100% blue at this point
kubectl run tmp2 --restart=Never --rm -i --image=busybox -n bluegreen -- wget -qO- --timeout=3 http://app-svc
# blue-response
```

Cut over — one `kubectl patch` on the Service's `selector`, nothing touched on either Deployment:
```bash
kubectl patch service app-svc -n bluegreen -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'
```
**Equally fast by hand:** `kubectl edit service app-svc -n bluegreen`, change `version: blue` to
`version: green` under `spec.selector`, save — this is a single scalar field, so `patch` and `edit`
are about the same effort either way; use whichever you're more comfortable typing under pressure.

Confirm the switch is total, not partial:
```bash
kubectl get endpoints app-svc -n bluegreen
# 3 IPs — all 3 belong to app-green Pods, zero from app-blue

kubectl run tmp3 --restart=Never --rm -i --image=busybox -n bluegreen -- wget -qO- --timeout=3 http://app-svc
# green-response
```

**Blue/green vs. canary, concretely:** both rely on the exact same mechanism — a Service selector
that can match one label value or another — but canary (`028`/`060`/`072`) achieves its ratio by
*replica count* under a selector that matches **both** versions simultaneously (`app: myapp` only,
no `version` in the Service's own selector), while blue/green's selector matches **exactly one**
`version` at a time and the switch is a single atomic patch — there's no in-between state where both
receive traffic. `app-blue`'s 3 Pods stay running, untouched, after the cutover — rolling back is
just patching the selector back to `blue`, instant, no rollout needed on either Deployment.

## Cleanup

```bash
kubectl delete ns bluegreen
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
