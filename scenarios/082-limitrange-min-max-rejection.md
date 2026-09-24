# 082 — A Pod rejected by a LimitRange's max and limit/request ratio, fixed to the largest limits allowed

**Domain:** Application Environment, Configuration and Security · **Difficulty:** Easy

`043` shows the *default-filling* side of a `LimitRange` on a Pod with no `resources:` block.
This is the other half: a Pod that does specify resources, but outside what the `LimitRange`
allows, gets rejected outright rather than adjusted. Two different gates fire at once here, and
only one of them is a plain maximum.

## Task

> The `palladium` team's Pod `pdf-worker`, defined in `~/ckad/082/pdf-worker.yaml`, is rejected
> when created in namespace `palladium`. Find out why. Then fix the manifest so the Pod is admitted
> with the **highest CPU and memory limits the namespace allows**, without changing its requests,
> and create it.

## Documentation

What to look up: **Limit Ranges**, the constraints part (`min`, `max`, `maxLimitRequestRatio`),
distinct from `default`/`defaultRequest`.
- <https://kubernetes.io/docs/concepts/policy/limit-range/> — same page as `043`, different section.

## Setup

```bash
kubectl create ns palladium

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: container-bounds
  namespace: palladium
spec:
  limits:
  - type: Container
    min:
      cpu: 50m
      memory: 32Mi
    max:
      cpu: "1"
      memory: 768Mi
    maxLimitRequestRatio:
      memory: "4"
EOF

mkdir -p ~/ckad/082
cat > ~/ckad/082/pdf-worker.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pdf-worker
  namespace: palladium
spec:
  containers:
  - name: pdf-worker
    image: busybox:1.36
    command: ["sh", "-c", "echo rendering; sleep 3600"]
    resources:
      requests:
        cpu: 250m
        memory: 96Mi
      limits:
        cpu: 1500m
        memory: 768Mi
EOF
```

## Solution

Reproduce the rejection:
```bash
kubectl apply -f ~/ckad/082/pdf-worker.yaml
```
```
Error from server (Forbidden): error when creating "/home/<you>/ckad/082/pdf-worker.yaml": pods "pdf-worker" is forbidden:
[maximum cpu usage per Container is 1, but limit is 1500m, memory max limit to request ratio per Container is 4, but provided ratio is 8.000000]
```
Unlike a ResourceQuota rejection (`052`, `058`), this happens with **no other Pods in the
namespace**: a `LimitRange` checks each *individual* container, not a running total. The memory limit
`768Mi` is exactly at the `max`, so the `max` gate is happy with it. What fails is the ratio:
`768Mi / 96Mi = 8`, and the namespace allows at most 4.

Read the `LimitRange` to get all the bounds at once:
```bash
kubectl describe limitrange container-bounds -n palladium
# Type       Resource  Min   Max    Default Request  Default Limit  Max Limit/Request Ratio
# ----       --------  ---   ---    ---------------  -------------  -----------------------
# Container  memory    32Mi  768Mi  768Mi            768Mi          4
# Container  cpu       50m   1      1                1              -
```
The `Default` columns are filled in although the Setup never set them: when a `LimitRange` has a
`max` but no `default`, the API server copies `max` into `default` (and `default` into
`defaultRequest`). A container with no `resources:` at all would therefore get `768Mi`/`1` as both
request and limit, which is rarely what anyone intended.

With the requests fixed, the highest limit allowed is the smaller of `max` and
`ratio × request`:
- CPU: no ratio for CPU, so the ceiling is `max` = `1` (`1000m`).
- memory: `min(768Mi, 4 × 96Mi)` = `min(768Mi, 384Mi)` = `384Mi`.

Fix the two limits in the file and create the Pod:
```bash
sed -i -e 's/cpu: 1500m/cpu: "1"/' -e 's/memory: 768Mi/memory: 384Mi/' ~/ckad/082/pdf-worker.yaml
kubectl apply -f ~/ckad/082/pdf-worker.yaml
kubectl wait --for=condition=Ready pod/pdf-worker -n palladium --timeout=60s
# pod/pdf-worker condition met

kubectl get pod pdf-worker -n palladium -o jsonpath='{.spec.containers[0].resources}{"\n"}'
# {"limits":{"cpu":"1","memory":"384Mi"},"requests":{"cpu":"250m","memory":"96Mi"}}
```
**Faster by hand:** open the file in `vi` and change the two numbers; the `sed` is only there so the
step can be pasted. A limit exactly *equal* to the `max` or to `ratio × request` is admitted: both
checks are "not greater than".

The `min` side rejects just as hard, for a *request* below it:
```bash
kubectl run tiny -n palladium --image=busybox:1.36 \
  --overrides='{"spec":{"containers":[{"name":"tiny","image":"busybox:1.36","command":["sleep","3600"],"resources":{"requests":{"cpu":"20m","memory":"16Mi"},"limits":{"cpu":"100m","memory":"64Mi"}}}]}}'
```
```
Error from server (Forbidden): pods "tiny" is forbidden: [minimum cpu usage per Container is 50m, but request is 20m,
minimum memory usage per Container is 32Mi, but request is 16Mi]
```

**Lesson:** a `LimitRange` is both a filler (`043`, when a field is *missing*) and a gate (here,
when a field is *present but out of bounds*). The gate has three parts, `min` on requests, `max` on
limits and `maxLimitRequestRatio` on the pair, and a limit can pass `max` and still fail the ratio.
The rejection message names each rule that failed, so read it before reaching for
`kubectl describe`.

## Cleanup

```bash
kubectl delete ns palladium
rm -rf ~/ckad/082
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
