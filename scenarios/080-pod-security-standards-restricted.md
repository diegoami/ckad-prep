# 080 — Pod Security Admission: preview, enforce and fix a Deployment under `restricted`

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

The other securityContext scenarios (`024`, `051`, `063`, `071`, `079`) work at the Pod level.
Pod Security Admission is the namespace-level layer that replaced PodSecurityPolicy. Turning it on
in a namespace that already has workloads is where it gets interesting: the running Pods are left
alone, and the damage only shows up the next time a controller tries to create a Pod.

## Task

> The `iridium` team's namespace `iridium` runs the Deployment `report-poller`. The platform team is
> moving namespaces to the `restricted` Pod Security Standard, pinned to the `v1.30` version of the
> standard.
>
> 1. Without changing anything yet, find out which of the Pods currently running in `iridium` would
>    violate it.
> 2. Enforce `restricted` (`v1.30`) on `iridium`, and also make it *warn* at the same level and
>    version.
> 3. Make `report-poller` compliant, so that it can roll out new Pods under the new policy.

## Documentation

What to look up: **Pod Security Standards**, and **Pod Security Admission**.
- <https://kubernetes.io/docs/concepts/security/pod-security-standards/> — the three levels
  (Privileged/Baseline/Restricted) and exactly which fields `restricted` requires.
- <https://kubernetes.io/docs/concepts/security/pod-security-admission/> — the
  `pod-security.kubernetes.io/<mode>` and `<mode>-version` namespace labels, and which modes apply
  to workload resources.
- <https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/> —
  the `--dry-run=server` preview.

## Setup

```bash
kubectl create ns iridium
kubectl create deployment report-poller -n iridium --image=busybox:1.36 --replicas=2 \
  -- sh -c 'while true; do echo polling reports; sleep 30; done'
kubectl rollout status deployment/report-poller -n iridium --timeout=60s
```

## Solution

Preview first. A server-side dry run of the label runs the admission check against every existing
Pod and prints the violations as warnings, without saving the label:
```bash
kubectl label --dry-run=server --overwrite ns iridium \
  pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=v1.30
```
```
Warning: existing pods in namespace "iridium" violate the new PodSecurity enforce level "restricted:v1.30"
Warning: report-poller-<hash>-<id> (and 1 other pod): allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
namespace/iridium labeled (server dry run)
```
Identical Pods are grouped (`and 1 other pod`), so both `report-poller` replicas are affected.
`--dry-run=client` would print nothing useful here: the check only happens in the API server.

Now apply it for real. Each mode has its own label and its own `-version` label, so that's four labels:
```bash
kubectl label --overwrite ns iridium \
  pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=v1.30 \
  pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/warn-version=v1.30
```
The real `label` prints the same two warnings, but violating Pods never block the label itself. No
CRD, controller or webhook to install: Pod Security Admission is built into the API server (stable
since 1.25), and the namespace labels are all it takes.

The running Pods are **not** evicted. Enforcement only happens when a Pod is created, which is why
this can look like it changed nothing:
```bash
kubectl get pods -n iridium
# both report-poller Pods still Running
```

Trigger a new rollout and watch it break. `enforce` checks only Pods, so the Deployment update
itself is accepted. The `warn` mode does check workload templates, and that's the only feedback you
get on the command line:
```bash
kubectl rollout restart deployment/report-poller -n iridium
# Warning: would violate PodSecurity "restricted:v1.30": allowPrivilegeEscalation != false (...), ...
# deployment.apps/report-poller restarted

sleep 5
kubectl get rs -n iridium
# the new ReplicaSet has DESIRED 1, CURRENT 0: it can't create any Pod

kubectl get events -n iridium --field-selector reason=FailedCreate -o custom-columns=MSG:.message | tail -1
# Error creating: pods "report-poller-<hash>-<id>" is forbidden: violates PodSecurity "restricted:v1.30": ...
```
Without the `warn` label, `rollout restart` prints nothing unusual and the rejection only shows up as
a `FailedCreate` event on the ReplicaSet. With a Deployment, always check the ReplicaSet (or
`kubectl get events`) when Pods don't appear.

Fix the Pod template. The rejection message lists every missing field at once, so there's no need to
remember the `restricted` rules. `busybox` runs as root by default, so `runAsNonRoot: true` also
needs a numeric non-root `runAsUser` (see `079`):
```bash
kubectl patch deployment report-poller -n iridium -p '
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: busybox
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
'
kubectl rollout status deployment/report-poller -n iridium --timeout=90s
# deployment "report-poller" successfully rolled out
```
The patch is a strategic merge, so the container is matched by its `name` (`busybox`, from the image
name, as `kubectl create deployment` chose it) and only the new fields are added. **Faster by
hand:** `kubectl edit deployment report-poller -n iridium` and add both `securityContext` blocks.
The rollout replaces the stuck ReplicaSet with a new one whose Pods pass admission:
```bash
kubectl get rs -n iridium
# NAME                       DESIRED   CURRENT   READY
# report-poller-<new-hash>   2         2         2      <- the compliant template
# report-poller-<hash>       0         0         0      <- the stuck one, scaled away
# report-poller-<hash>       0         0         0      <- the original
```

Note where each field lives: `runAsNonRoot`, `runAsUser` and `seccompProfile` can be set at the
**Pod** level (`spec.securityContext`), where they act as defaults for every container, while
`allowPrivilegeEscalation` and `capabilities` exist only per **container**
(`spec.containers[].securityContext`). The rejection message hints at this ("pod or container"
versus "container"), but it's easy to miss: put `capabilities` under the Pod's `securityContext` and
the API server rejects it as an unknown field.

Pinning `-version` to `v1.30` keeps the rules fixed when the cluster is upgraded. Without it the
label means `latest`, and the rules can tighten under you after an upgrade.

## Cleanup

```bash
kubectl delete ns iridium
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
