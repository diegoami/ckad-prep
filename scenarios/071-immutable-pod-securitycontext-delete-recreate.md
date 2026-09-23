# 071 — Add securityContext fields to a running Pod that can't be `kubectl edit`ed in place

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`051` and `063` patch a **Deployment**, which works because the controller replaces the Pods for
you. This one is a **bare Pod** with no controller. The same field change is rejected on a running
Pod, so you have to dump, edit, delete and recreate it yourself.

## Task

> In namespace `matterhorn`, the Pod `report-generator` runs on its own, without a Deployment or
> any other controller. Security review requires that it runs as user `30001` and that its
> container `generator` can't escalate privileges. Make the change so that a Pod named
> `report-generator` is running with both settings, and keep the manifest you used at
> `~/ckad/071/report-generator.yaml`.

## Documentation

What to look up: **Pods** (which fields are immutable after creation), plus **Security Context**.
- <https://kubernetes.io/docs/concepts/workloads/pods/#pod-update-and-replacement> — most of a
  Pod's spec can't be changed in place once created, which is why this scenario has to
  delete and recreate.
- <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>

## Setup

```bash
kubectl create ns matterhorn

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: report-generator
  namespace: matterhorn
spec:
  containers:
  - name: generator
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
EOF

kubectl wait --for=condition=Ready pod/report-generator -n matterhorn --timeout=60s
```

## Solution

First see why an in-place change doesn't work. Try patching `securityContext` on the running Pod:
```bash
kubectl patch pod report-generator -n matterhorn -p '{"spec":{"securityContext":{"runAsUser":30001}}}'
```
```
The Pod "report-generator" is invalid: spec: Forbidden: pod updates may not change fields other
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
kubectl get pod report-generator -n matterhorn -o yaml > ~/ckad/071/report-generator.yaml
```

**Gotcha, and the trap this scenario is built around:** the dumped YAML already contains an
*empty* `securityContext: {}` under `spec` (the API server always fills it in, even when nothing was
set). If you *insert* a new `securityContext:` block above the containers list instead of editing
the one that's already there, the document ends up with **two** `securityContext:` keys at the same
level under `spec`. That duplicate mapping key isn't rejected: `kubectl apply` doesn't error, and
the parser silently keeps only the *later* one. A block inserted *before* the existing empty one is
thrown away, and `runAsUser` never lands. Tested directly: a manifest with `securityContext:
{runAsUser: 30001}` followed by the original `securityContext: {}` applies cleanly (a server-side
dry run confirms it on v1.30) and comes back with an empty `spec.securityContext`. The fix looks as
if it worked and didn't.

The correct edit **replaces** the existing empty block instead of adding a second one:
```yaml
spec:
  securityContext:      # this key already exists in the dump — edit it, don't duplicate it
    runAsUser: 30001    # add
  containers:
  - command: [ ... ]
    image: busybox:1.28
    name: generator
    securityContext:            # add — the container had none, safe to add fresh here
      allowPrivilegeEscalation: false   # add
```

Delete and recreate. The stale `uid`, `resourceVersion` and `status` in the dump don't matter: the
API server ignores them on a create, as `060` also shows:
```bash
kubectl delete pod report-generator -n matterhorn --force --grace-period=0
kubectl apply -f ~/ckad/071/report-generator.yaml
kubectl wait --for=condition=Ready pod/report-generator -n matterhorn --timeout=60s
```

**Faster by hand:** `kubectl edit pod report-generator -n matterhorn`, make the same two edits and
save. The API server rejects the change, but kubectl saves your edited copy to a temp file and
prints its path (`/tmp/kubectl-edit-....yaml`). `kubectl replace --force -f <that file>` then
deletes and recreates the Pod in one step. Copy the file to `~/ckad/071/` if the task wants the
manifest kept.

Confirm both fields landed and are enforced at runtime:
```bash
kubectl get pod report-generator -n matterhorn -o jsonpath='{.spec.securityContext.runAsUser}{"\n"}{.spec.containers[0].securityContext.allowPrivilegeEscalation}{"\n"}'
# 30001
# false

kubectl exec -n matterhorn report-generator -- id
# uid=30001 gid=0(root) groups=0(root)
```

**Lesson:** when hand-editing a `kubectl get -o yaml` dump, search for a field name before adding
it. A Pod's spec comes back with several empty-but-present blocks (`securityContext: {}` chief
among them) that look like nothing is there, and inserting a second key with the same name
produces a silently broken duplicate rather than a visible error.

## Cleanup

```bash
kubectl delete ns matterhorn
rm -rf ~/ckad/071
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
