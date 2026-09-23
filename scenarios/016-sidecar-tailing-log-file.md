# 016 — Add a native sidecar container that tails a shared log file

**Domain:** Application Design and Build · **Difficulty:** Hard

## Task

> The `opal` team runs a Deployment `audit-writer` in namespace `opal`. Its only regular container,
> `writer`, appends audit events to `/var/log/audit/audit.log` on a shared `emptyDir` volume, so
> nothing shows up in `kubectl logs`. Add a sidecar container named `log-shipper`, image
> `busybox:1.36`, that mounts the same volume and follows `audit.log` with `tail -f`, so the audit
> events can be read with `kubectl logs ... -c log-shipper`. Don't change the existing containers.

## Documentation

What to look up: **Sidecar Containers**.
- <https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/> — native sidecars are
  init containers with `restartPolicy: Always`; this page covers the shared-volume pattern directly.

## Setup

The Deployment exists without the sidecar; adding it is the task.

```bash
kubectl create ns opal
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audit-writer
  namespace: opal
spec:
  replicas: 2
  selector:
    matchLabels: {app: audit-writer}
  template:
    metadata:
      labels: {app: audit-writer}
    spec:
      volumes:
      - name: audit-logs
        emptyDir: {}
      initContainers:
      - name: prepare-log
        image: busybox:1.36
        command: ['sh', '-c', 'echo "audit log opened" > /var/log/audit/audit.log']
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/audit
      containers:
      - name: writer
        image: busybox:1.36
        command: ['sh', '-c', 'while true; do echo "$(date): user=svc-batch action=read" >> /var/log/audit/audit.log; sleep 1; done']
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/audit
EOF
kubectl -n opal rollout status deployment audit-writer --timeout=60s
```

## Ordering gotcha

A *native* sidecar (Kubernetes 1.29+) is an `initContainers` entry with `restartPolicy: Always`.
Unlike a regular init container it doesn't have to finish: the kubelet starts it and moves on, and
it keeps running next to the main containers. The sidecar and the main container can therefore
race each other. If the sidecar's `tail -f audit.log` starts before anything has created
`audit.log`, `tail` exits straight away with `No such file or directory`, and the Pod stays in
`Init:Error` / `Init:CrashLoopBackOff`.

The Setup avoids that with the **regular** init container `prepare-log`, which creates the file
and exits. Init containers run in list order, so put the sidecar **after** `prepare-log`. The file
then already exists when the sidecar starts. If you put the sidecar first, you get the race back.

## Solution

```bash
kubectl -n opal edit deployment audit-writer
```

Append the sidecar to `initContainers`, after `prepare-log`:

```yaml
      initContainers:
      - name: prepare-log
        # ...unchanged...
      - name: log-shipper                                            # add
        image: busybox:1.36                                          # add
        restartPolicy: Always                                        # add — makes this a sidecar
        command: ['sh', '-c', 'tail -f /var/log/audit/audit.log']    # add
        volumeMounts:                                                # add
        - name: audit-logs                                           # add
          mountPath: /var/log/audit                                  # add
```

The same change as a single copy-pasteable command:

```bash
kubectl -n opal patch deployment audit-writer --type=json -p='[
  {"op":"add","path":"/spec/template/spec/initContainers/-","value":{
    "name":"log-shipper","image":"busybox:1.36","restartPolicy":"Always",
    "command":["sh","-c","tail -f /var/log/audit/audit.log"],
    "volumeMounts":[{"name":"audit-logs","mountPath":"/var/log/audit"}]}}]'
```

Verify:

```bash
kubectl -n opal rollout status deployment audit-writer --timeout=60s
# pick the newest Pod: right after the rollout, old Pods can still be Terminating
POD=$(kubectl -n opal get pods -l app=audit-writer --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].metadata.name}')
kubectl -n opal get pod "$POD"
# READY 2/2 — the sidecar counts toward pod readiness, unlike a regular init container

kubectl -n opal logs "$POD" -c log-shipper --tail=5
# ... user=svc-batch action=read   (a new line every second)
```

## Cleanup

```bash
kubectl delete ns opal
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
