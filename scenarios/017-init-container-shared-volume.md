# 017 — Init container that populates a shared volume before the main container starts

**Domain:** Application Design and Build · **Difficulty:** Medium

See also `guide/exam-tips.md` § "Init Containers — write paths must be absolute", which covers the
trap this scenario is built around.

## Task

> In namespace `garnet`, the Deployment `docs-site` runs one `nginx:1.25-alpine` Pod that serves
> files from an `emptyDir` volume mounted at nginx's web root. The volume starts out empty, so the
> site has no content. Add an init container named `seed-content`, image `busybox:1.36`, that
> writes a file `index.html` containing `welcome to garnet docs` to the root of that volume before
> nginx starts. Confirm the page is served by requesting it from a temporary Pod.

## Documentation

What to look up: **Init Containers**.
- <https://kubernetes.io/docs/concepts/workloads/pods/init-containers/> — ordering guarantees
  (run to completion, in sequence, before any app container starts) and the shared-volume pattern.

## Setup

The Deployment exists; the init container doesn't yet.

```bash
kubectl create ns garnet
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docs-site
  namespace: garnet
spec:
  replicas: 1
  selector:
    matchLabels: {app: docs-site}
  template:
    metadata:
      labels: {app: docs-site}
    spec:
      volumes:
      - name: site-root
        emptyDir: {}
      containers:
      - image: nginx:1.25-alpine
        name: nginx
        volumeMounts:
        - name: site-root
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
EOF
kubectl -n garnet rollout status deployment docs-site --timeout=60s
```

## Solution — and the trap to notice

The obvious-looking command is wrong:

```yaml
command: ['sh', '-c', 'echo "welcome to garnet docs" > index.html']
```

This writes to the init container's own working directory (`/`), **not** into the mounted
volume. The file never reaches the shared `emptyDir`, and nothing in the Pod's status tells you
why: the Pod comes up `Running`/`Ready`, and nginx just answers `403 Forbidden` for an empty
directory. Use the absolute path of the mount instead:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docs-site
  namespace: garnet
spec:
  replicas: 1
  selector:
    matchLabels: {app: docs-site}
  template:
    metadata:
      labels: {app: docs-site}
    spec:
      volumes:
      - name: site-root
        emptyDir: {}
      initContainers:                 # add
      - name: seed-content            # add
        image: busybox:1.36           # add
        command: ['sh', '-c', 'echo "welcome to garnet docs" > /usr/share/nginx/html/index.html']  # add — absolute path
        volumeMounts:                 # add
        - name: site-root             # add
          mountPath: /usr/share/nginx/html   # add
      containers:
      - image: nginx:1.25-alpine
        name: nginx
        volumeMounts:
        - name: site-root
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
EOF

kubectl -n garnet rollout status deployment docs-site --timeout=60s

# newest Pod: the old one may still be Terminating right after the rollout
POD_IP=$(kubectl -n garnet get pods -l app=docs-site --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1:].status.podIP}')
kubectl -n garnet run tmp --restart=Never --rm -i --image=nginx:alpine -- \
  curl -s -m 5 "http://$POD_IP"
# welcome to garnet docs
```

## Cleanup

```bash
kubectl delete ns garnet
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
