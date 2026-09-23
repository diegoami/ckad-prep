# 080 — Pod Security Standards: a namespace label rejecting non-compliant Pods

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

The other securityContext scenarios (`024`, `051`, `063`, `071`, `079`) work at the Pod level.
Pod Security Admission is the namespace-level layer that replaced PodSecurityPolicy, and it rejects
Pods before any of those settings get a chance to matter.

## Task

> Make namespace `secure` enforce the `restricted` Pod Security Standard, so it rejects any Pod
> that doesn't meet it. Confirm a plain Pod gets rejected, then create one that actually satisfies the
> standard.

## Documentation

What to look up: **Pod Security Standards**, and **Pod Security Admission**.
- <https://kubernetes.io/docs/concepts/security/pod-security-standards/> — the three levels
  (Privileged/Baseline/Restricted) and exactly which fields `restricted` requires.
- <https://kubernetes.io/docs/concepts/security/pod-security-admission/> — the
  `pod-security.kubernetes.io/enforce` namespace label mechanism itself.

## Setup

```bash
kubectl create ns secure
```

## Solution

Turn on enforcement with a namespace label:
```bash
kubectl label ns secure pod-security.kubernetes.io/enforce=restricted
```
No CRD, no extra controller, no admission webhook to install — Pod Security Admission has been a
built-in, always-on part of the API server since Kubernetes 1.25; the namespace label alone is
enough to activate it.

Confirm the rejection — a completely ordinary Pod, no securityContext at all:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: secure
spec:
  containers:
  - name: web
    image: nginx:1.25-alpine
EOF
```
```
Error from server (Forbidden): error when creating "STDIN": pods "web" is forbidden:
violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "web" must
set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "web"
must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "web"
must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "web" must set
securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```
This is the single most useful thing about Pod Security Admission for the exam: the rejection
message itself enumerates every field still missing, all at once — no need to consult the
`restricted` spec from memory or guess field names one at a time.

Build a Pod satisfying all four, reusing `079`'s unprivileged nginx image (a `runAsNonRoot`-safe
image is a prerequisite here too — `restricted` requires it just like `079`'s task did on its own):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: secure
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: web
    image: nginxinc/nginx-unprivileged:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
EOF
```
```bash
kubectl get pod web -n secure
# READY 1/1, STATUS Running
```

Note where each field lives: `runAsNonRoot` and `seccompProfile` can be set at the **Pod** level
(`spec.securityContext`), where they act as defaults for every container in the Pod, while
`allowPrivilegeEscalation` and `capabilities.drop` exist only per-**container**
(`spec.containers[].securityContext`). The rejection message hints at this ("pod or container"
versus "container"), but it's easy to miss: put `capabilities` under the Pod's `securityContext`
and the API server rejects it as an unknown field.

**Difference from `enforce=restricted` worth knowing:** `pod-security.kubernetes.io/warn=restricted`
(a separate label) only prints a warning on `kubectl apply` without blocking creation — useful for
auditing an existing namespace before actually turning on `enforce` and breaking anything already
running there.

## Cleanup

```bash
kubectl delete ns secure
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23. Pod Security
Admission behaves the same on v1.30 as on newer versions, including the rejection message.*
