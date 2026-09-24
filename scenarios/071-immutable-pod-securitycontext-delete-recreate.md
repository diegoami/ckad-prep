# 071 — Harden a bare Pod whose securityContext can't be changed in place

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`051` and `063` patch a **Deployment**, which works because the controller replaces the Pods for
you. This one is a **bare Pod** with no controller. The same field change is rejected on a running
Pod, so you have to dump, edit, delete and recreate it yourself.

## Task

> The Fornax observability team runs a single Pod `telemetry-agent` in namespace `fornax`. It has no
> Deployment or other controller behind it. A hardening review asks for two changes:
>
> - every process in the Pod runs with user ID `4210` and primary group ID `4210`;
> - the container `collector` gets a read-only root filesystem.
>
> Afterwards a Pod named `telemetry-agent` must be running with these settings. Save the manifest
> you used as `~/ckad/071/telemetry-agent.yaml`.

## Documentation

What to look up: **Pods** (which fields are immutable after creation), plus **Security Context**.
- <https://kubernetes.io/docs/concepts/workloads/pods/#pod-update-and-replacement> — most of a
  Pod's spec can't be changed in place once created, which is why this scenario has to
  delete and recreate.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>

## Setup

```bash
kubectl create ns fornax

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: telemetry-agent
  namespace: fornax
  labels:
    app: telemetry-agent
spec:
  containers:
  - name: collector
    image: busybox:1.36
    command: ["sh", "-c", "while true; do date >> /tmp/heartbeat; sleep 30; done"]
EOF

kubectl wait --for=condition=Ready pod/telemetry-agent -n fornax --timeout=60s
```

## Solution

First see why an in-place change doesn't work. Try patching `securityContext` on the running Pod:
```bash
kubectl patch pod telemetry-agent -n fornax -p '{"spec":{"securityContext":{"runAsUser":4210,"runAsGroup":4210}}}'
```
```
The Pod "telemetry-agent" is invalid: spec: Forbidden: pod updates may not change fields other
than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,
`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds`
(allow it to be set to 1 if it was previously negative)
```
A bare Pod's spec is almost entirely immutable once created. Only that short allow-list of fields
can change on a live Pod (notably `image`, not `securityContext`). `kubectl edit` hits exactly the
same rejection; there's no special edit-mode bypass.

Dump the Pod, edit the YAML, delete, recreate:
```bash
mkdir -p ~/ckad/071
kubectl get pod telemetry-agent -n fornax -o yaml > ~/ckad/071/telemetry-agent.yaml
```

**Gotcha, and the trap this scenario is built around:** the dumped YAML already contains an
*empty* `securityContext: {}` under `spec` (the API server always fills it in, even when nothing was
set). If you *insert* a new `securityContext:` block above the containers list instead of editing
the one that's already there, the document ends up with **two** `securityContext:` keys at the same
level under `spec`. That duplicate mapping key isn't rejected: `kubectl apply` doesn't error, and
the parser silently keeps only the *later* one. A block inserted *before* the existing empty one is
thrown away, and `runAsUser`/`runAsGroup` never land. A server-side dry run shows it:
```bash
sed 's/^  containers:$/  securityContext:\n    runAsUser: 4210\n    runAsGroup: 4210\n  containers:/' \
    ~/ckad/071/telemetry-agent.yaml \
  | kubectl apply --dry-run=server -f - -o jsonpath='{.spec.securityContext}{"\n"}'
# {}
```
The dry run even *succeeds* against the still-running Pod, because once the duplicate is dropped
nothing has changed. The fix looks as if it worked and didn't.

The correct edit **replaces** the existing empty block instead of adding a second one:
```yaml
spec:
  securityContext:      # this key already exists in the dump — edit it, don't duplicate it
    runAsUser: 4210     # add
    runAsGroup: 4210    # add
  containers:
  - command: [ ... ]
    image: busybox:1.36
    name: collector
    securityContext:              # add — the container had none, safe to add fresh here
      readOnlyRootFilesystem: true   # add
```
Pod-level `securityContext` holds `runAsUser`/`runAsGroup` (they'd also be accepted per container),
but `readOnlyRootFilesystem` only exists at **container** level. Put it under `spec.securityContext`
and the API server rejects the Pod with `strict decoding error: unknown field
"spec.securityContext.readOnlyRootFilesystem"`.

Scripted version of the same edit, for pasting (on the exam you'd make it in `vim`). The
`/^spec:/,/^status:/` range keeps the second substitution away from the `name: collector` line
under `status.containerStatuses`:
```bash
sed -i -e 's/^  securityContext: {}$/  securityContext:\n    runAsUser: 4210\n    runAsGroup: 4210/' \
       -e '/^spec:/,/^status:/ s/^    name: collector$/    name: collector\n    securityContext:\n      readOnlyRootFilesystem: true/' \
       ~/ckad/071/telemetry-agent.yaml
```

Delete and recreate. The stale `uid`, `resourceVersion` and `status` in the dump don't matter: the
API server ignores them on a create, as `060` also shows:
```bash
kubectl delete pod telemetry-agent -n fornax --force --grace-period=0
kubectl apply -f ~/ckad/071/telemetry-agent.yaml
kubectl wait --for=condition=Ready pod/telemetry-agent -n fornax --timeout=60s
```

**Gotcha #2:** the new Pod reaches `Running` but the heartbeat loop is now broken. With a read-only
root filesystem, `date >> /tmp/heartbeat` fails, because `/tmp` is part of the image's filesystem:
```bash
kubectl exec -n fornax telemetry-agent -- sh -c 'echo x > /tmp/heartbeat'
# sh: can't create /tmp/heartbeat: Read-only file system
# command terminated with exit code 1
```
The shell loop keeps going, so nothing crashes; it just silently stops writing. The task only asks
for the two settings, so this is where it ends here. For an app that must write somewhere, the usual
answer is an `emptyDir` volume mounted at that path (here `/tmp`): volumes stay writable under
`readOnlyRootFilesystem`, and you'd add it in the same edit, since `volumes` is immutable on a
running Pod too.

**Faster by hand:** `kubectl edit pod telemetry-agent -n fornax`, make the same edits and save.
The API server rejects the change, but kubectl saves your edited copy to a temp file and prints its
path (`/tmp/kubectl-edit-....yaml`). `kubectl replace --force -f <that file>` then deletes and
recreates the Pod in one step. Copy the file to `~/ckad/071/` if the task wants the manifest kept.

Confirm the fields landed and are enforced at runtime:
```bash
kubectl get pod telemetry-agent -n fornax -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[0].securityContext}{"\n"}'
# {"runAsGroup":4210,"runAsUser":4210}
# {"readOnlyRootFilesystem":true}

kubectl exec -n fornax telemetry-agent -- id
# uid=4210 gid=4210 groups=4210
```

**Lesson:** when hand-editing a `kubectl get -o yaml` dump, search for a field name before adding
it. A Pod's spec comes back with several empty-but-present blocks (`securityContext: {}` chief
among them) that look like nothing is there, and inserting a second key with the same name
produces a silently broken duplicate rather than a visible error.

## Cleanup

```bash
kubectl delete ns fornax
rm -rf ~/ckad/071
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
