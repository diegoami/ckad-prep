# 077 — Size a rollout's blast radius with maxSurge/maxUnavailable, push a bad tag, read the history, roll back

**Domain:** Application Deployment · **Difficulty:** Medium

`044` tunes the strategy with a healthy image throughout, and `066` is a rollback with no strategy
tuning. This one chains both: the strategy decides exactly how many Pods a bad release can take down
before it stalls, and the image change is one that can never become ready, so `rollout status`,
`rollout history` and `rollout undo` all have something to show.

## Task

> In namespace `zinc`, the `checkout` Deployment runs 8 replicas of `httpd:2.4.62-alpine`.
>
> 1. Change its rolling-update strategy so a rollout may run at most 3 Pods above the desired count
>    and may take at most 1 Pod out of service at a time. Use absolute numbers, not percentages.
> 2. Release image `httpd:2.4.99-alpine`, and record the change cause `release httpd 2.4.99` so it
>    shows up in the rollout history.
> 3. The release won't come up. Before fixing anything, work out how many Pods of each revision
>    exist while it's stuck, and confirm the Deployment still has 7 Pods available.
> 4. Put `checkout` back on the revision it ran before the release, with all 8 Pods ready.

## Documentation

What to look up: **Deployments**: rolling update strategy, rollout history and rollback.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment>
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#checking-rollout-history-of-a-deployment>
  — the `kubernetes.io/change-cause` annotation.
- <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment>

## Setup

```bash
kubectl create ns zinc
kubectl create deployment checkout -n zinc --image=httpd:2.4.62-alpine --replicas=8
kubectl rollout status deployment/checkout -n zinc --timeout=90s
```

## Solution

Set the strategy. Both fields live under `spec.strategy.rollingUpdate` and accept either an integer
or a percentage *string*. Here they're plain integers:
```bash
kubectl patch deployment checkout -n zinc --type=merge \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":3,"maxUnavailable":1}}}}'
kubectl get deployment checkout -n zinc -o jsonpath='{.spec.strategy}{"\n"}'
# {"rollingUpdate":{"maxSurge":3,"maxUnavailable":1},"type":"RollingUpdate"}
```
**Faster by hand:** `kubectl edit deployment checkout -n zinc` and change the two values under
`spec.strategy.rollingUpdate`. Writing `"3"` in quotes there is not the same thing: a string must be
a percentage, and the API server rejects it.

Release the new image. `kubectl create deployment` names the container after the image's
repository, so it's `httpd`, not `checkout`. Check it rather than guess:
```bash
kubectl get deploy checkout -n zinc -o jsonpath='{.spec.template.spec.containers[0].name}{"\n"}'
# httpd

kubectl set image deployment/checkout httpd=httpd:2.4.99-alpine -n zinc
kubectl annotate deployment checkout -n zinc kubernetes.io/change-cause="release httpd 2.4.99"
```
`--record` used to fill in the change cause automatically, but it's deprecated. The annotation is
the supported way, and the Deployment controller copies it onto the new revision's ReplicaSet,
which is where `rollout history` reads it from.

Watch the rollout stall:
```bash
kubectl rollout status deployment/checkout -n zinc --timeout=20s
# Waiting for deployment "checkout" rollout to finish: 4 out of 8 new replicas have been updated...
# error: timed out waiting for the condition

kubectl get rs -n zinc -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,DESIRED:.spec.replicas,READY:.status.readyReplicas
# NAME                  IMAGE                 DESIRED   READY
# checkout-<hash>       httpd:2.4.62-alpine   7         7
# checkout-<hash>       httpd:2.4.99-alpine   4         <none>

kubectl get deployment checkout -n zinc
# NAME       READY   UP-TO-DATE   AVAILABLE   AGE
# checkout   7/8     4            7           ...

kubectl get pods -n zinc | grep -c ImagePull
# 4
```
The numbers come straight from the strategy. `maxUnavailable: 1` lets the old ReplicaSet drop from 8
to 7. `maxSurge: 3` caps the total at `8 + 3 = 11`, so the new ReplicaSet gets `11 - 7 = 4` Pods.
None of them become ready (the tag doesn't exist, so they sit in `ErrImagePull`/`ImagePullBackOff`),
so the old ReplicaSet can't be scaled down any further and the rollout stalls there. The blast
radius of a bad release is `maxUnavailable` Pods of lost capacity, and `maxSurge` only decides how
many broken Pods get created alongside.

Check the history. The stalled release still got its own revision:
```bash
kubectl rollout history deployment/checkout -n zinc
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         release httpd 2.4.99

kubectl rollout history deployment/checkout -n zinc --revision=1 | grep Image
#     Image:	httpd:2.4.62-alpine
```

Roll back. With only two revisions, a plain `undo` goes to revision 1; `--to-revision=1` says the
same thing explicitly, which is safer once there are more revisions (see `066`):
```bash
kubectl rollout undo deployment/checkout -n zinc --to-revision=1
kubectl rollout status deployment/checkout -n zinc --timeout=90s
# deployment "checkout" successfully rolled out

kubectl get deployment checkout -n zinc -o jsonpath='{.spec.template.spec.containers[0].image} {.status.readyReplicas}/{.spec.replicas}{"\n"}'
# httpd:2.4.62-alpine 8/8

kubectl rollout history deployment/checkout -n zinc
# REVISION  CHANGE-CAUSE
# 2         release httpd 2.4.99
# 3         <none>
```
Rolling back doesn't restore "revision 1": it re-applies revision 1's Pod template, which becomes
revision 3, and revision 1 disappears from the list. The Deployment's `change-cause` annotation is
cleared along the way, so revision 3 shows `<none>` and doesn't carry the failed release's text over.

## Cleanup

```bash
kubectl delete ns zinc
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
