# 045 — Copy a file into and out of a running container

**Domain:** Application Observability and Maintenance · **Difficulty:** Easy

`kubectl cp` is a different tool from `exec`, `logs` and `debug` — copying a file doesn't require
a shell in the target at all.

## Task

> Copy a local file into Pod `cptest` (namespace `cpns`) at `/tmp/cptest.txt`, confirm it landed,
> then copy a file back out of the container to the host (`/etc/nginx/nginx.conf` from inside the
> Pod) and confirm you have it locally.

## Documentation

What to look up: `kubectl cp` in the **kubectl Reference Docs**.
- <https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#cp> — the
  `<namespace>/<pod>:<path>` source/destination syntax in both directions, and that it requires `tar`
  inside the container.

## Setup

```bash
kubectl create ns cpns
kubectl -n cpns run cptest --image=nginx:1.25-alpine
kubectl -n cpns wait --for=condition=ready pod/cptest --timeout=30s
echo "hello from host" > cptest.txt
```

## Solution

```bash
# host -> container
kubectl -n cpns cp cptest.txt cptest:/tmp/cptest.txt
kubectl -n cpns exec cptest -- cat /tmp/cptest.txt
# hello from host

# container -> host
kubectl -n cpns cp cptest:/etc/nginx/nginx.conf ./nginx.conf.local
head -3 nginx.conf.local
```

`kubectl cp` syntax is `<namespace>/<pod>:<path>` on whichever side is the container — omitting the
namespace prefix and relying on `-n` (as above) works the same way. It shells out to `tar` inside
the container under the hood (confirmed live: stderr shows `tar: removing leading '/' from member
names` on the container->host direction) — so it fails on a distroless/scratch image with no
`tar` binary, even though the image might otherwise have everything else needed. For a
tar-less target, `kubectl exec ... -- cat <file>` redirected to a local file is the fallback.

## Cleanup

```bash
kubectl delete ns cpns
rm -f cptest.txt nginx.conf.local
```
