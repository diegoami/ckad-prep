# 013 — StorageClass with an unfulfilled provisioner, capture the pending reason

**Domain:** Application Design and Build · **Difficulty:** Medium

## Task

> The storage team is rolling out a new archive tier for the `spruce` team, but their provisioner
> isn't deployed yet. Prepare the Kubernetes side now:
>
> - Create a StorageClass named `spruce-archive` that uses provisioner
>   `archive.example.com/spruce` and keeps volumes after their claim is deleted
>   (`reclaimPolicy: Retain`).
> - In Namespace `spruce`, create a PersistentVolumeClaim named `audit-logs-pvc` that uses this
>   StorageClass and requests 5Gi with access mode `ReadWriteOnce`.
>
> Since no provisioner is running, the claim won't bind. Save the event message that explains why
> it is still waiting to `~/ckad/013/pvc-pending-reason`.

## Documentation

What to look up: **Storage Classes**.
- <https://kubernetes.io/docs/concepts/storage/storage-classes/> — provisioners, and why a PVC
  referencing a StorageClass with no working provisioner stays `Pending` instead of erroring.

## Setup

```bash
kubectl create ns spruce
mkdir -p ~/ckad/013
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: spruce-archive
provisioner: archive.example.com/spruce
reclaimPolicy: Retain
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audit-logs-pvc
  namespace: spruce
spec:
  storageClassName: spruce-archive
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

kubectl -n spruce get pvc audit-logs-pvc
# STATUS: Pending. Expected, nothing implements archive.example.com/spruce

kubectl -n spruce describe pvc audit-logs-pvc | grep -A3 Events:
```

The event reads:

```
Events:
  Type    Reason                Age   From                         Message
  ----    ------                ----  ----                         -------
  Normal  ExternalProvisioning  ...   persistentvolume-controller  Waiting for a volume to be created either by the external provisioner 'archive.example.com/spruce' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
```

Save it. Either the `describe` snippet or just the message works; the message alone is cleaner:

```bash
kubectl -n spruce get events --field-selector involvedObject.name=audit-logs-pvc \
  -o jsonpath='{.items[0].message}{"\n"}' > ~/ckad/013/pvc-pending-reason
cat ~/ckad/013/pvc-pending-reason
# Waiting for a volume to be created either by the external provisioner 'archive.example.com/spruce' ...
```

If you don't set `volumeBindingMode` it defaults to `Immediate`, which is why the controller tries
to provision as soon as the claim exists. With `WaitForFirstConsumer` (as on kind's `standard`
class) the PVC would instead wait for a Pod to use it, with a different event message.

## Cleanup

```bash
kubectl delete ns spruce
kubectl delete storageclass spruce-archive
rm -rf ~/ckad/013
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
