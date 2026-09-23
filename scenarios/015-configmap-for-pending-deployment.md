# 015 — Create the missing ConfigMap a Deployment is already waiting on

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

## Task

> The `willow` team deployed their marketing site as the Deployment `landing-page` in Namespace
> `willow`, but its Pod never starts. The Deployment expects its web content to come from a
> ConfigMap named `landing-page-html`, which nobody created. Create that ConfigMap from the file
> `~/ckad/015/landing.html`, storing the content under the key `index.html`. Then check with an
> HTTP request from a temporary Pod that the site serves the new page.

## Documentation

What to look up: **ConfigMaps**.
- <https://kubernetes.io/docs/concepts/configuration/configmap/> — creating one and referencing
  it from a Pod spec (env var vs. volume, same shape as Secrets in `014`).

## Setup

The Deployment already references the ConfigMap in its volume, so its Pod sits in
`ContainerCreating` until the ConfigMap exists.

```bash
kubectl create ns willow
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: landing-page
  namespace: willow
spec:
  replicas: 1
  selector:
    matchLabels: {app: landing-page}
  template:
    metadata:
      labels: {app: landing-page}
    spec:
      volumes:
      - name: content
        configMap:
          name: landing-page-html
      containers:
      - name: web
        image: nginx:1.27-alpine
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
EOF

kubectl -n willow get pods
# STATUS: ContainerCreating

mkdir -p ~/ckad/015
cat > ~/ckad/015/landing.html <<'EOF'
<!DOCTYPE html>
<html><head><title>Willow Outdoor Gear</title></head>
<body>Spring collection now in stock.</body></html>
EOF
```

## Solution

```bash
# the Pod's events name the missing ConfigMap
kubectl -n willow describe pod -l app=landing-page | tail -3
# Warning  FailedMount  ...  MountVolume.SetUp failed for volume "content" : configmap "landing-page-html" not found

kubectl -n willow create configmap landing-page-html --from-file=index.html=$HOME/ckad/015/landing.html

kubectl -n willow rollout status deployment landing-page --timeout=120s
kubectl -n willow get pods
# now Running. The kubelet retries the mount and picks up the ConfigMap by itself

POD=$(kubectl -n willow get pods -l app=landing-page -o jsonpath='{.items[0].metadata.name}')
kubectl -n willow exec "$POD" -- ls /usr/share/nginx/html
# index.html

POD_IP=$(kubectl -n willow get pod "$POD" -o jsonpath='{.status.podIP}')
kubectl run tmp --restart=Never --rm -i --image=nginx:alpine -n willow -- curl -s -m 5 "$POD_IP"
# <!DOCTYPE html>
# <html><head><title>Willow Outdoor Gear</title></head>
# <body>Spring collection now in stock.</body></html>
```

The `index.html=` prefix in `--from-file` sets the key name. Without it, the key would be the file
name (`landing.html`), and nginx would answer `403 Forbidden` for `/` because there's no `index.html`.

You don't need to delete the Pod. The kubelet keeps retrying the failed mount (with a growing
back-off), so the Pod starts on its own once the ConfigMap exists. That took a few seconds on a
fresh setup, but can take longer if the Pod had already been failing for a while.

## Cleanup

```bash
kubectl delete ns willow
rm -rf ~/ckad/015
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
