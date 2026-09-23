# 021 — Deployment running under a specific ServiceAccount with memory requests/limits

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

## Task

> The reporting team works in namespace `onyx`. Create a Deployment `report-gen` there with 3
> replicas of image `httpd:2.4-alpine`, and name the container `report-worker`. Each container
> should request `32Mi` of memory and be limited to `64Mi`. The Pods must run under the
> ServiceAccount `onyx-reports-sa`, which already exists in the namespace.

## Documentation

What to look up: **Managing Resources for Containers**, plus **ServiceAccounts**.
- <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/> — `requests`
  vs. `limits` field placement under `spec.containers[].resources`.
- <https://kubernetes.io/docs/concepts/security/service-accounts/> — `spec.serviceAccountName` on a Pod template.

## Setup

```bash
kubectl create ns onyx
kubectl -n onyx create serviceaccount onyx-reports-sa
mkdir -p ~/ckad/021
```

## Solution

```bash
kubectl -n onyx create deployment report-gen --replicas=3 --image=httpd:2.4-alpine \
  --dry-run=client -o yaml > ~/ckad/021/report-gen.yaml
```

Edit the generated YAML (`vi ~/ckad/021/report-gen.yaml`):

```yaml
spec:
  template:
    spec:
      serviceAccountName: onyx-reports-sa   # add
      containers:
      - image: httpd:2.4-alpine
        name: report-worker                 # was: httpd
        resources:                          # was: resources: {}
          requests:                         # add
            memory: 32Mi                    # add
          limits:                           # add
            memory: 64Mi                    # add
```

Apply and check:

```bash
kubectl apply -f ~/ckad/021/report-gen.yaml
kubectl -n onyx rollout status deployment report-gen --timeout=60s
kubectl -n onyx get pods -l app=report-gen
# 3 Pods, 1/1 Running

kubectl -n onyx get pods -l app=report-gen \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.serviceAccountName}{"  "}{.spec.containers[0].name}{"  "}{.spec.containers[0].resources}{"\n"}{end}'
# report-gen-...  onyx-reports-sa  report-worker  {"limits":{"memory":"64Mi"},"requests":{"memory":"32Mi"}}
```

`kubectl set serviceaccount deployment report-gen onyx-reports-sa` and
`kubectl set resources deployment report-gen --requests=memory=32Mi --limits=memory=64Mi` also
work on an existing Deployment. They can't rename the container, though, so writing the YAML is
still the quickest route here.

## Cleanup

```bash
kubectl delete ns onyx
rm -rf ~/ckad/021
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
