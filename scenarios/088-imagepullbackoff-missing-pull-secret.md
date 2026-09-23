# 088 — ImagePullBackOff from a private registry, fixed with `imagePullSecrets`

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Medium

`011` and `068` push *to* a registry but never pull from one that requires credentials. This one
uses the `docker-registry` Secret type and the `imagePullSecrets` field to fix a Pod stuck in
`ImagePullBackOff`.

## Task

> Pod `secret-app` in namespace `registryns` references an image on a private registry. Diagnose
> why it won't start, then fix it. The registry credentials are user `testuser`, password
> `testpass123`.

## Documentation

What to look up: **Pull an Image from a Private Registry**.
- <https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/> — the
  `kubectl create secret docker-registry` walkthrough and `imagePullSecrets` field, end to end.

## Setup

On the exam the private registry already exists. Locally you build one: the same disposable
`registry:2` pattern as `011`/`068`, but with HTTP basic auth enabled, so an unauthenticated pull
genuinely fails the way it would against any real private registry. Run all Setup blocks in the
same shell, since later ones reuse `$REG_IP` and `$NODE`.

```bash
mkdir -p ~/ckad/088/auth
docker run --rm httpd:2.4-alpine htpasswd -Bbn testuser testpass123 > ~/ckad/088/auth/htpasswd

docker run -d -p 5050:5050 --name auth-registry \
  -v ~/ckad/088/auth:/auth \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5050 \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2

docker network connect kind auth-registry
REG_IP=$(docker inspect auth-registry --format '{{.NetworkSettings.Networks.kind.IPAddress}}')
# the kind node's container has the same name as the Kubernetes node
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
```

Push a test image with real credentials:
```bash
docker pull busybox:1.36
docker login localhost:5050 -u testuser -p testpass123
docker tag busybox:1.36 localhost:5050/private/secret-app:v1
docker push localhost:5050/private/secret-app:v1
docker logout localhost:5050
```

**Local-lab quirk:** a kind node's containerd defaults to HTTPS for any registry host, so pulling
from a plain-HTTP local registry (`registry:2` has no TLS out of the box) fails with
`server gave HTTP response to HTTPS client` — *before* auth is even checked. Tell containerd to use
plain HTTP for this one registry, so the real error shows up. This isn't part of the lesson and
has no exam equivalent:
```bash
cat > ~/ckad/088/hosts.toml <<EOF
server = "http://$REG_IP:5050"

[host."http://$REG_IP:5050"]
  capabilities = ["pull", "resolve"]
EOF
docker exec "$NODE" mkdir -p "/etc/containerd/certs.d/$REG_IP:5050"
docker cp ~/ckad/088/hosts.toml "$NODE:/etc/containerd/certs.d/$REG_IP:5050/hosts.toml"

# Older kind releases (v0.23) don't point containerd at /etc/containerd/certs.d at all, so the
# hosts.toml above would be ignored. Add the setting if it's missing, keeping a backup for Cleanup.
if ! docker exec "$NODE" grep -q config_path /etc/containerd/config.toml; then
  docker exec "$NODE" cp /etc/containerd/config.toml /etc/containerd/config.toml.bak-088
  docker exec "$NODE" sh -c 'printf "\n[plugins.\"io.containerd.grpc.v1.cri\".registry]\n  config_path = \"/etc/containerd/certs.d\"\n" >> /etc/containerd/config.toml'
fi
docker exec "$NODE" systemctl restart containerd
```

Create the Pod the task starts from:
```bash
kubectl create ns registryns
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-app
  namespace: registryns
spec:
  containers:
  - name: secret-app
    image: $REG_IP:5050/private/secret-app:v1
    command: ["sh", "-c", "sleep 3600"]
EOF
```

## Solution

```bash
kubectl get pod secret-app -n registryns
# STATUS: ErrImagePull, then ImagePullBackOff

kubectl describe pod secret-app -n registryns | tail -3
# Failed to pull image "...": ... pull access denied, repository does not exist or may require
# authorization: authorization failed: no basic auth credentials
```
"pull access denied ... may require authorization" is the specific phrase that distinguishes a
missing-credentials problem from a genuinely wrong image name/tag (which would instead say
`manifest unknown` or `not found`) — read the exact wording before assuming which one it is.

Create a `docker-registry`-type Secret — the dedicated generator, not `--from-literal`. The
`--docker-server` value must match the registry host in the image reference exactly:
```bash
REG_IP=$(docker inspect auth-registry --format '{{.NetworkSettings.Networks.kind.IPAddress}}')
kubectl create secret docker-registry regcred \
  --docker-server="$REG_IP:5050" \
  --docker-username=testuser \
  --docker-password=testpass123 \
  -n registryns
```
On the exam you'd read the registry host straight off the Pod's `image:` field instead.

**Gotcha:** `imagePullSecrets` can't be added to an existing Pod — patching it in place is
rejected outright, like `securityContext` in `071`:
```bash
kubectl patch pod secret-app -n registryns -p '{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}'
# The Pod "secret-app" is invalid: spec: Forbidden: pod updates may not change fields other than ...
```
`kubectl edit` isn't a workaround here either — same as `071`, it submits the identical update on
save and hits the identical rejection. Delete and recreate with the field present from the start:
```bash
kubectl delete pod secret-app -n registryns
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-app
  namespace: registryns
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: secret-app
    image: $REG_IP:5050/private/secret-app:v1
    command: ["sh", "-c", "sleep 3600"]
EOF
```

Confirm:
```bash
kubectl wait --for=condition=Ready pod/secret-app -n registryns --timeout=60s
kubectl describe pod secret-app -n registryns | grep -A5 Events:
# Normal   Pulled   ...   Successfully pulled image "..." — no more auth failure
```

On a Deployment (unlike a bare Pod), the same fix is a routine patch — `imagePullSecrets` lives
under `spec.template.spec`, so a `kubectl patch deployment` there triggers a normal rolling update,
no delete/recreate dance required, exactly like every other Pod-template field.

## Cleanup

```bash
kubectl delete ns registryns
REG_IP=$(docker inspect auth-registry --format '{{.NetworkSettings.Networks.kind.IPAddress}}')
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
docker exec "$NODE" rm -rf "/etc/containerd/certs.d/$REG_IP:5050"
docker exec "$NODE" sh -c '[ -f /etc/containerd/config.toml.bak-088 ] && mv /etc/containerd/config.toml.bak-088 /etc/containerd/config.toml'
docker exec "$NODE" systemctl restart containerd
docker rm -f auth-registry
rm -rf ~/ckad/088
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
