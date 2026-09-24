# 067 — Revive an old CronJob manifest whose seccomp setting silently stopped working

**Domain:** Application Observability and Maintenance · **Difficulty:** Medium

In `031` and `076` the old manifest is rejected until every problem is fixed. Here only the
apiVersion is rejected. The second problem, a seccomp annotation from before the `seccompProfile`
field existed, is accepted with nothing more than a warning, and it no longer does anything.

## Task

> The `amchur` team kept the manifest for its nightly `invoice-archiver` CronJob in
> `~/ckad/067/invoice-archiver.yaml`; it was last used on a much older cluster. Get it running on
> this cluster from that file, updated in place:
>
> - it must use the API version this cluster serves for CronJobs;
> - security policy requires the Pods it creates to run under the container runtime's default
>   seccomp profile, and that has to be in effect in the running container, not just declared;
> - `kubectl apply -f` on the final file must print no warnings.
>
> Prove the seccomp requirement with a run started by hand.

## Documentation

What to look up: **Deprecated API Migration Guide**, plus **Configure a Security Context**.
- <https://kubernetes.io/docs/reference/using-api/deprecation-guide/#cronjob-v125> — CronJob
  `batch/v1beta1` is no longer served as of v1.25; move to `batch/v1`.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-seccomp-profile-for-a-container>
  — the `seccompProfile` field and `type: RuntimeDefault`.

## Setup

```bash
kubectl create ns amchur
mkdir -p ~/ckad/067
cat > ~/ckad/067/invoice-archiver.yaml <<'EOF'
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: invoice-archiver
  namespace: amchur
spec:
  schedule: "30 2 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          annotations:
            seccomp.security.alpha.kubernetes.io/pod: runtime/default
        spec:
          restartPolicy: OnFailure
          containers:
          - name: archiver
            image: busybox:1.36
            command: ["sh", "-c", "grep Seccomp: /proc/self/status; echo archiving invoices"]
EOF
```
The container prints its own seccomp mode, which makes the result easy to check: `0` means no
seccomp filter, `2` means a filter is loaded.

## Solution

First attempt, as it is:
```bash
cd ~/ckad/067
kubectl apply -f invoice-archiver.yaml
# error: resource mapping not found for name: "invoice-archiver" namespace: "amchur" from
# "invoice-archiver.yaml": no matches for kind "CronJob" in version "batch/v1beta1"
# ensure CRDs are installed first
```
`batch/v1beta1` CronJobs stopped being served in v1.25. The migration guide lists no field changes
for CronJob, so the version line is all this API needs:
```bash
sed -i 's#^apiVersion: batch/v1beta1#apiVersion: batch/v1#' invoice-archiver.yaml
kubectl apply -f invoice-archiver.yaml
# Warning: spec.jobTemplate.spec.template.metadata.annotations[seccomp.security.alpha.kubernetes.io/pod]:
#   non-functional in v1.27+; use the "seccompProfile" field instead
# cronjob.batch/invoice-archiver created
```
It applied, so it's tempting to stop here. The warning says the annotation is ignored, and a run
shows it:
```bash
kubectl create job archiver-check-1 -n amchur --from=cronjob/invoice-archiver
# Warning: spec.template.metadata.annotations[seccomp.security.alpha.kubernetes.io/pod]: non-functional ...
kubectl wait --for=condition=Complete job/archiver-check-1 -n amchur --timeout=60s
kubectl logs job/archiver-check-1 -n amchur
# Seccomp:	0
# archiving invoices
```
`0`: the container runs with no seccomp filter at all, which is what the old annotation was meant
to prevent.

The replacement is the `seccompProfile` field in a `securityContext`. The old annotation applied to
the whole Pod, so the matching place is the Pod-level `securityContext` in the Job template.
Remove the annotation and add the field. Edit the file (the result should look like this):
```bash
cat > invoice-archiver.yaml <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: invoice-archiver
  namespace: amchur
spec:
  schedule: "30 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          securityContext:
            seccompProfile:
              type: RuntimeDefault
          restartPolicy: OnFailure
          containers:
          - name: archiver
            image: busybox:1.36
            command: ["sh", "-c", "grep Seccomp: /proc/self/status; echo archiving invoices"]
EOF

kubectl apply -f invoice-archiver.yaml
# cronjob.batch/invoice-archiver configured      <- no Warning line this time
```
Removing the annotation from the file is enough: `kubectl apply` remembers what it applied last
time and deletes fields that have disappeared from the file.

Prove it with a new run. Jobs copy the template when they are created, so the first check Job
still has the old settings:
```bash
kubectl create job archiver-check-2 -n amchur --from=cronjob/invoice-archiver
kubectl wait --for=condition=Complete job/archiver-check-2 -n amchur --timeout=60s
kubectl logs job/archiver-check-2 -n amchur
# Seccomp:	2
# archiving invoices

kubectl get cronjob invoice-archiver -n amchur \
  -o jsonpath='{.spec.jobTemplate.spec.template.metadata.annotations}{"\n"}{.spec.jobTemplate.spec.template.spec.securityContext}{"\n"}'
# (empty line: the annotation is gone)
# {"seccompProfile":{"type":"RuntimeDefault"}}
```

Takeaways:
- An apiVersion removal fails loudly. A deprecated field or annotation often doesn't: the object is
  accepted and the only sign is a `Warning:` line from the API server. Read those lines.
- The seccomp annotations (`seccomp.security.alpha.kubernetes.io/pod` and
  `container.seccomp.security.alpha.kubernetes.io/<name>`) date from before the field existed. On
  current clusters they are ignored, so a manifest that still uses them runs unconfined unless the
  node's kubelet applies `RuntimeDefault` by default.
- `type: RuntimeDefault` on the Pod applies to every container. Set it on one container's
  `securityContext` instead if only that container should get it.

## Cleanup

```bash
cd ~
kubectl delete ns amchur
rm -rf ~/ckad/067
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
