# 037 — Triage a Deployment broken by two unrelated, stacked bugs

**Domain:** Application Observability and Maintenance · **Difficulty:** Hard

Most exam questions surface one bug at a time; the real exam sometimes stacks several, where
fixing the first only reveals the next.

## Task

> Deployment `broken-app` in namespace `triagens` has never reached `Running`. Find out why and fix
> it — completely; don't stop at the first error you see.

## Documentation

What to look up: **Troubleshoot Applications** (the Debug Pods / Debug Running Pods pages).
- <https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/> — the general
  Pending/CrashLoopBackOff/ImagePullBackOff triage flow this scenario stacks three ways.

## Setup

```bash
kubectl create ns triagens
kubectl -n triagens create configmap app-cfg --from-literal=LOG_LEVEL=info

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: triagens
spec:
  replicas: 1
  selector: {matchLabels: {app: broken-app}}
  template:
    metadata: {labels: {app: broken-app}}
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "0"]
        env:
        - name: MISSING_KEY
          valueFrom:
            configMapKeyRef:
              name: app-cfg
              key: DOES_NOT_EXIST
        resources:
          limits:
            memory: "10Mi"
EOF
```

## Solution

Two bugs, revealed one at a time.

```bash
kubectl -n triagens get pods
# STATUS: CreateContainerConfigError
kubectl -n triagens describe pod -l app=broken-app | grep -A3 Events:
# couldn't find key DOES_NOT_EXIST in ConfigMap triagens/app-cfg
```

**Bug 1 — env var references a ConfigMap key that doesn't exist.** The container can't even be
created until this resolves, which is why nothing past this point is visible yet.

```bash
kubectl -n triagens get configmap app-cfg -o yaml   # confirm what key(s) actually exist — LOG_LEVEL
kubectl -n triagens patch deployment broken-app --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/env/0/valueFrom/configMapKeyRef/key","value":"LOG_LEVEL"}
]'
```

```bash
kubectl -n triagens get pods
# STATUS: OOMKilled (then CrashLoopBackOff) — a *new* failure, only visible now that the
# container actually started
```

**Bug 2 — the memory limit (10Mi) is far below what this workload actually needs** (it deliberately
allocates 50M). Fixing bug 1 alone gets you to a *different* error, not to `Running` — exactly
the "fix one thing, find the next" pattern this scenario exists to drill.

```bash
kubectl -n triagens patch deployment broken-app --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"100Mi"}
]'

kubectl -n triagens get pods
# STATUS: Running — confirmed live, both bugs actually fixed, not just the first one
```

The general lesson: after any fix, re-check status — don't assume the *next* symptom is the same
root cause as the last one. `kubectl describe pod` and `kubectl get pods` after every change is
cheap; assuming you're done after one fix is the actual trap here.

## Cleanup

```bash
kubectl delete ns triagens
```
