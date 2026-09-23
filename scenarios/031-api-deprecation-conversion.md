# 031 — Convert a manifest off a removed API version

**Domain:** Application Observability and Maintenance · **Difficulty:** Medium

Complements `drills/drill.md` 7.9, which drills the `kubectl-convert` one-liner but has nothing
to actually convert.

## Task

> An old Ingress manifest was written against `extensions/v1beta1`, which no longer exists on this
> cluster's API server. Get it running again on the current stable API version.

## Documentation

What to look up: **Deprecated API Migration Guide**.
- <https://kubernetes.io/docs/reference/using-api/deprecation-guide/> — lists every removed API
  version by Kubernetes release, including exactly the `extensions/v1beta1` → `networking.k8s.io/v1`
  Ingress move this scenario does by hand.

## Setup

```bash
kubectl create ns convertns
kubectl -n convertns create deployment web --image=httpd:2.4-alpine --port=80
kubectl -n convertns expose deployment web --port=80

cat > old-ingress.yaml <<'EOF'
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: old-ingress
  namespace: convertns
spec:
  rules:
  - host: web.ckad.test
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 80
EOF
```

## Solution

```bash
# confirm it's actually rejected — this is why the task exists, not a hypothetical
kubectl apply -f old-ingress.yaml
# error: resource mapping not found for name: "old-ingress" ... no matches for kind "Ingress"
# in version "extensions/v1beta1"
```

If `kubectl-convert` is installed: `kubectl convert -f old-ingress.yaml --output-version
networking.k8s.io/v1 -o yaml > new-ingress.yaml` does this automatically. But `kubectl convert` is
a separate plugin, not bundled with `kubectl` itself — without it you get `error: unknown command
"convert"`, and you can't assume it's installed on the exam. Manual conversion is the more broadly
useful skill.

Converting by hand, an `extensions/v1beta1` → `networking.k8s.io/v1` Ingress always needs two
structural changes: `spec.rules[].http.paths[].backend` goes from flat `serviceName`/`servicePort`
fields to a nested `service.name`/`service.port.number` object, and every path now requires an
explicit `pathType`.

```bash
cat > new-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: old-ingress
  namespace: convertns
spec:
  ingressClassName: nginx
  rules:
  - host: web.ckad.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
EOF

kubectl apply -f new-ingress.yaml
# ingress.networking.k8s.io/old-ingress created — proves the conversion is correct,
# not just plausible-looking
```

## Cleanup

```bash
kubectl delete ns convertns
rm -f old-ingress.yaml new-ingress.yaml
```
