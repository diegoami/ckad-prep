# 005 — Helm release operations (uninstall, upgrade, install with values, clean up stuck release)

**Domain:** Application Deployment · **Difficulty:** Medium

## Task

> The `falcon` team manages their billing services with Helm, all in Namespace `falcon`. Tidy up
> their releases:
> - Remove the release `billing-api-v1`; it has been retired.
> - `billing-api-v2` is running an outdated chart version. Upgrade it to a newer version of the
>   same chart.
> - Install a new release called `billing-docs` from an Apache chart. Its Deployment must run 2
>   replicas, and that number must be set through Helm values at install time, not by scaling
>   afterwards.
> - One release in the namespace never finished installing and is stuck in `pending-install`.
>   Find out which one it is and remove it.

## Documentation

What to look up: **Helm** release lifecycle commands.
- <https://helm.sh/docs/intro/using_helm/> — install/upgrade/uninstall walkthrough, including `--set`.
- <https://helm.sh/docs/helm/helm_rollback/> — the dedicated `helm rollback` command reference.

## Setup

The exam provides its own chart repository. Locally, Bitnami's public `nginx` and `apache` charts
play that role — same shape, same `--set` values pattern. Chart versions move over time, so if
`--version 25.1.9` below no longer exists, pick any older version from
`helm search repo bitnami/nginx --versions`.

The last command manufactures the stuck release: it starts an install that waits for a Pod which
can never be scheduled, then kills the `helm` process before it can record a result. The release
is left in `pending-install`, exactly as it would be if someone's terminal had died mid-install.

```bash
kubectl create ns falcon
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update bitnami

helm -n falcon install billing-api-v1 bitnami/nginx
helm -n falcon install billing-api-v2 bitnami/nginx --version 25.1.9
helm -n falcon install billing-portal bitnami/nginx

timeout -s KILL 15 helm -n falcon install billing-worker bitnami/nginx \
  --set-string nodeSelector.pool=reserved --wait --timeout 10m || true
helm -n falcon ls
```

## Solution

```bash
# 1. remove the retired release
helm -n falcon uninstall billing-api-v1

# 2. upgrade billing-api-v2 to a newer chart version
helm -n falcon history billing-api-v2
# REVISION  STATUS    CHART         ...
# 1         deployed  nginx-25.1.9
helm search repo bitnami/nginx --versions | head -5
helm -n falcon upgrade billing-api-v2 bitnami/nginx     # no --version = latest in the repo
helm -n falcon ls
# billing-api-v2  ...  2  deployed  nginx-25.1.14 (or whatever is newest)

# 3. install the Apache chart with 2 replicas set via values
helm show values bitnami/apache | grep -A1 'replicaCount:'
# replicaCount: 1
helm -n falcon install billing-docs bitnami/apache --set replicaCount=2
kubectl -n falcon get deploy billing-docs-apache
# NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
# billing-docs-apache   0/2     2            0           5s

# 4. find and remove the stuck release
helm -n falcon ls --pending
# NAME            ...  STATUS           CHART
# billing-worker  ...  pending-install  nginx-25.1.14
helm -n falcon uninstall billing-worker
helm -n falcon ls
# billing-api-v2, billing-docs, billing-portal — all deployed
```

Gotchas:

- **Finding pending releases.** In Helm 3, plain `helm ls` shows only `deployed` and `failed`
  releases, so a `pending-install` release is invisible unless you add `-a`/`--all` or
  `--pending`. Helm 4 lists every status by default and dropped the `-a` shorthand. `--pending`
  works in both, so it's the safe habit.
- **Finding the right value key.** Don't guess what a chart calls its replica setting. Run
  `helm show values <chart> | grep -i replica` and use the exact key.
- **Chart name vs. release name.** The Deployment is called `billing-docs-apache` (release name
  plus chart name), not `billing-docs`. Look it up with `kubectl get deploy` rather than assuming.
- **Apache Pods stuck in `ImagePullBackOff` locally.** In 2025 Bitnami stopped publishing most of
  its versioned images to the free `docker.io/bitnami` namespace, so the Apache chart's Pods may
  never start on your kind cluster. That doesn't affect this task: what's being checked is the
  release and the Deployment's `2` desired replicas (`UP-TO-DATE 2`). The nginx charts use the
  `latest` tag, which is still available, so those Pods do run.

## Cleanup

```bash
kubectl delete ns falcon
helm repo remove bitnami   # skip if you use the bitnami repo for anything else
```

---

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
