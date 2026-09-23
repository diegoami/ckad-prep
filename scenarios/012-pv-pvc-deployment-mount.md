# 012 — PersistentVolume, matching PersistentVolumeClaim, and a Deployment that mounts it

**Domain:** Application Design and Build · **Difficulty:** Medium

## Task

> The `maple` team's `report-archiver` service needs storage that outlives its Pods.
>
> - Create a PersistentVolume named `maple-reports-pv` with 1Gi capacity, access mode
>   `ReadWriteOnce`, backed by hostPath `/mnt/maple-reports`. It must not use a StorageClass.
> - In Namespace `maple`, create a PersistentVolumeClaim named `maple-reports-pvc` that requests
>   1Gi with access mode `ReadWriteOnce`, also without a StorageClass. It must bind to
>   `maple-reports-pv`.
> - In Namespace `maple`, create a Deployment named `report-archiver` with image `httpd:2.4-alpine`
>   that mounts the claim at `/var/reports`.

## Gotcha specific to practicing on `kind`

`kind` ships a default StorageClass (`standard`, via `local-path-provisioner`). If you leave
`storageClassName` out of the PVC, the admission controller quietly fills in `standard`. The PV
you created by hand has no class at all, so the two never match, and the PVC sits `Pending` with no
error explaining why. **Set `storageClassName: ""` explicitly on both** to opt out of the default
and force static binding. On a cluster with a default class, that's what "no StorageClass" has to
mean. Confirmed live: omitting it leaves the PVC `Pending`; setting `""` on both binds immediately.

## Documentation

What to look up: **Persistent Volumes** — the PV/PVC binding lifecycle.
- <https://kubernetes.io/docs/concepts/storage/persistent-volumes/> — how a PVC binds to a
  matching PV, and which fields (`capacity`, `accessModes`, `storageClassName`) have to line up.

## Setup

```bash
kubectl create ns maple
```

## Solution

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: maple-reports-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  hostPath:
    path: "/mnt/maple-reports"
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maple-reports-pvc
  namespace: maple
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  volumeName: maple-reports-pv   # optional; pins the claim to this PV
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pv maple-reports-pv
kubectl -n maple get pvc maple-reports-pvc
# both should show STATUS Bound

mkdir -p ~/ckad/012 && cd ~/ckad/012
kubectl -n maple create deploy report-archiver --image=httpd:2.4-alpine --dry-run=client -o yaml > report-archiver.yaml
```

Edit `report-archiver.yaml` to add the volume and mount:

```yaml
spec:
  template:
    spec:
      volumes:                            # add
      - name: reports                     # add
        persistentVolumeClaim:            # add
          claimName: maple-reports-pvc    # add
      containers:
      - image: httpd:2.4-alpine
        name: httpd
        volumeMounts:                     # add
        - name: reports                   # add
          mountPath: /var/reports         # add
```

```bash
kubectl -n maple apply -f report-archiver.yaml
kubectl -n maple rollout status deployment report-archiver --timeout=60s
POD=$(kubectl -n maple get pods -l app=report-archiver -o jsonpath='{.items[0].metadata.name}')
kubectl -n maple describe pod "$POD" | grep -A2 Mounts:
#     Mounts:
#       /var/reports from reports (rw)
```

## Cleanup

```bash
kubectl delete ns maple
kubectl delete pv maple-reports-pv
cd ~ && rm -rf ~/ckad/012
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
