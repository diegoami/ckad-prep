# 003 — Create a Pod and a reusable status-check script

**Domain:** Core kubectl · **Difficulty:** Easy

## Task

> In Namespace `wren`, start a single Pod called `intranet-web` from the image `httpd:2.4-alpine`.
> Its container must be named `web`.
>
> The `wren` team lead wants a one-liner they can run by hand whenever they need to know the
> current status of that specific Pod. Put that command in `~/ckad/003/intranet-web-status.sh`. It
> has to use `kubectl`.

## Documentation

What to look up: **Pod lifecycle** (the `phase` field this script checks) and `jsonpath` output.
- <https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/> — the `Pending`/`Running`/
  `Succeeded`/`Failed`/`Unknown` phase values.
- <https://kubernetes.io/docs/reference/kubectl/jsonpath/> — jsonpath syntax for `kubectl get -o jsonpath`.

## Setup

```bash
kubectl create ns wren
mkdir -p ~/ckad/003
```

## Solution

`kubectl run` always names the container after the Pod, so generate the YAML and fix the container
name before creating anything:

```bash
kubectl -n wren run intranet-web --image=httpd:2.4-alpine --dry-run=client -o yaml \
  > ~/ckad/003/intranet-web.yaml
```

Edit the generated YAML, renaming the container:

```yaml
spec:
  containers:
  - image: httpd:2.4-alpine
    name: web   # was: intranet-web
```

```bash
kubectl apply -f ~/ckad/003/intranet-web.yaml
kubectl -n wren wait --for=condition=ready pod/intranet-web --timeout=60s
kubectl -n wren get pod intranet-web -o jsonpath='{.spec.containers[*].name}{"\n"}'
# web

echo 'kubectl -n wren get pod intranet-web -o jsonpath="{.status.phase}"' \
  > ~/ckad/003/intranet-web-status.sh
chmod +x ~/ckad/003/intranet-web-status.sh
bash ~/ckad/003/intranet-web-status.sh
# Running
```

Put `-n wren` inside the script. Whoever runs it later may have a different default namespace
set, and a script without it would then report `NotFound`. `kubectl -n wren get pod intranet-web`
or a `describe ... | grep Status` would also satisfy "output the status"; the jsonpath version is
just the tidiest.

## Cleanup

```bash
kubectl delete ns wren
rm -rf ~/ckad/003
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
