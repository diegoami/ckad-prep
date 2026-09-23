# CKAD Drill Sheet

> **How to use this deck:** short questions in 19 sections, each answered by one command or a
> short YAML snippet.
>
> 1. Read a question and answer it from memory: type the command into a terminal (ideally
>    against a practice cluster) or write the YAML out. Don't peek.
> 2. Expand **answer** and compare. Where it matters, the answer also explains *why* and points out
>    the trap.
> 3. If you got it wrong, were unsure, or had to look something up, put an `x` next to the heading.
>    At the end of each session, go back over every `x`. Remove an `x` once you've got that
>    question right in two later sessions.
>
> [drills/workbook.md](workbook.md) has the same questions with the answers left blank. Copy it
> and write your own answers there, then check them against this file.
>
> Each heading links to the kubernetes.io (or helm/podman) page you're allowed to open in the
> exam. Learn where those pages are as well as what's on them.

---

## Setup — run this at the start of every practice session

```bash
alias k=kubectl                              # usually already set up in the exam terminal
export do="--dry-run=client -o yaml"         # k create deploy x --image=nginx $do > x.yaml
export now="--force --grace-period=0"        # k delete pod x $now
```

The answers below use `k`, `$do` and `$now` to keep them short while you practise. In the exam
every question runs on a fresh VM (see [my exam setup](../guide/exam-tips.md#my-exam-setup)), so
exported variables don't carry over: type the flags out, or export them again per question.

---

## 1. Core kubectl

> **Docs:** [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

### 1.1 Create an nginx pod in namespace `dev`, never restart — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run nginx --image=nginx --restart=Never -n dev
```
</details>

---

### 1.2 Generate the YAML for the pod above without creating it — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run nginx --image=nginx --restart=Never -n dev $do
```

`$do` expands to `--dry-run=client -o yaml`. Always write `--dry-run=client` in full: a bare
`--dry-run` is deprecated, and `--dry-run=server` sends the object to the API server for validation
without saving it.
</details>

---

### 1.3 Create a busybox pod that runs `env`, prints output, then auto-deletes — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run busybox --image=busybox --restart=Never -it --rm -- env
```

- `--restart=Never` makes a one-shot Pod. With the default (`Always`) the container would be
  restarted after `env` exits.
- `--rm` only works when your terminal is attached to the container. `-i` alone is enough (`-t` just
  adds a TTY, which a non-interactive `env` doesn't need). With neither, kubectl refuses:
  `error: --rm should only be used for attached containers`.
</details>

---

### 1.4 Get all pods across all namespaces, wide output — [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get po -A -o wide
```
</details>

---

### 1.5 Change the image of pod `nginx` container `nginx` to `nginx:1.24.0` — [kubectl set image](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_image/)
<details><summary>answer</summary>

```bash
k set image pod/nginx nginx=nginx:1.24.0
```
</details>

---

### 1.6 Get the pod IP of `nginx` using jsonpath — [JSONPath support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
<details><summary>answer</summary>

```bash
k get pod nginx -o jsonpath='{.status.podIP}'
```
</details>

---

### 1.7 Execute a shell inside a running pod `nginx` — [kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
<details><summary>answer</summary>

```bash
k exec -it nginx -- /bin/sh
```
</details>

---

### 1.8 Get logs from the previous (crashed) instance of pod `nginx` — [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
<details><summary>answer</summary>

```bash
k logs nginx -p
```
</details>

---

### 1.9 Create a ResourceQuota `myrq` with 1 CPU, 1G memory, 2 pods max (dry run only) — [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
<details><summary>answer</summary>

```bash
k create quota myrq --hard=cpu=1,memory=1G,pods=2 $do
```
</details>

---

### 1.10 Find where `livenessProbe` lives under a Deployment's spec, in one command — [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
<details><summary>answer</summary>

```bash
k explain deployment.spec.template.spec.containers.livenessProbe
```

Chain the full dotted path directly instead of drilling `explain deployment`, then
`explain deployment.spec`, etc. one level at a time.
</details>

---

### 1.11 You remember a Pod spec has a `capabilities` field somewhere, but not its exact location — find it without opening the docs — [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
<details><summary>answer</summary>

```bash
k explain pod.spec --recursive | grep -i capabilities
```

`--recursive` dumps the whole subtree in one call; piping to `grep -i` turns `explain` into a fast
field-finder when you know the name but not the path. The indentation of each match shows how deep
it is (here `containers[].securityContext.capabilities`, repeated for `initContainers` and
`ephemeralContainers`). For a resource with multiple API versions,
pin one explicitly: `k explain networkpolicy.spec --api-version=networking.k8s.io/v1`.
</details>

---

### 1.12 You forgot the exact flag to roll back a Deployment to a specific revision — find it in 2 seconds without leaving the terminal — [kubectl rollout -h](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_undo/)
<details><summary>answer</summary>

```bash
k rollout undo -h
```

`kubectl explain` describes resource *fields*; `-h` on the verb itself describes the *command's
flags and usage examples* — `explain` won't tell you `--to-revision` exists. Reach for `-h` on
rusty imperative one-liners (`set image`, `autoscale`, `taint`, `cordon`/`drain`) instead of
guessing or reconstructing syntax from memory.
</details>

---

### 1.13 Create a ResourceQuota that caps *requests* at 1 CPU/1Gi and *limits* at 2 CPU/2Gi separately — [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
<details><summary>answer</summary>

```bash
k create quota myrq2 --hard=requests.cpu=1,requests.memory=1Gi,limits.cpu=2,limits.memory=2Gi $do
```

Different from 1.9. There, `cpu`/`memory` are shorthand for `requests.cpu`/`requests.memory`, so
that quota caps only requests and leaves limits unbounded. Setting `requests.*` and `limits.*`
separately lets a namespace burst above what it's guaranteed, which suits a shared namespace better.
Once a quota covers a resource, every new Pod in the namespace must set that request or limit (or get
one from a LimitRange), or it's rejected.
</details>

---

## 2. Labels & Annotations

> **Docs:** [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

### 2.1 Create 3 nginx pods with label `app=v1` in one line — [Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
<details><summary>answer</summary>

```bash
for i in 1 2 3; do k run nginx$i --image=nginx -l app=v1; done
```
</details>

---

### 2.2 Show all pods with label columns `app` and `tier` — [Label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)
<details><summary>answer</summary>

```bash
k get po -L app,tier
```
</details>

---

### 2.3 Change label `app=v1` to `app=v2` on pod `nginx2` — [kubectl label](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/)
<details><summary>answer</summary>

```bash
k label po nginx2 app=v2 --overwrite
```
</details>

---

### 2.4 Get pods where `app=v2` but NOT `tier=frontend` — [Set-based requirements](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)
<details><summary>answer</summary>

```bash
k get po -l 'app=v2,tier!=frontend'
```
</details>

---

### 2.5 Add label `tier=web` to all pods with `app=v1` or `app=v2` — [Set-based requirements](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)
<details><summary>answer</summary>

```bash
k label po -l 'app in (v1,v2)' tier=web
```
</details>

---

### 2.6 Remove the `app` label from pods `nginx1 nginx2 nginx3` — [kubectl label](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/)
<details><summary>answer</summary>

```bash
k label po nginx1 nginx2 nginx3 app-
# or
k label po -l app app-
```
</details>

---

### 2.7 Annotate pods `nginx1 nginx2` with `owner=marketing` — [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
<details><summary>answer</summary>

```bash
k annotate po nginx1 nginx2 owner=marketing
```
</details>

---

## 3. Deployments

> **Docs:** [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

### 3.1 Create a deployment `web` with image `nginx:1.18.0`, 3 replicas, port 80 — [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
<details><summary>answer</summary>

```bash
k create deploy web --image=nginx:1.18.0 --replicas=3 --port=80
```
</details>

---

### 3.2 Update the image to `nginx:1.19.8` — [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
<details><summary>answer</summary>

```bash
k set image deploy web nginx=nginx:1.19.8
```
</details>

---

### 3.3 Check rollout status, then see history — [Deployment rollout](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#checking-rollout-history-of-a-deployment)
<details><summary>answer</summary>

```bash
k rollout status deploy web
k rollout history deploy web
```
</details>

---

### 3.4 Undo the last rollout — [Rolling back](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
<details><summary>answer</summary>

```bash
k rollout undo deploy web
```
</details>

---

### 3.5 Roll back to revision 2 specifically — [Rolling back to a specific revision](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-to-a-previous-revision)
<details><summary>answer</summary>

```bash
k rollout undo deploy web --to-revision=2
```
</details>

---

### 3.6 Pause the rollout, change image, resume — [Pausing and resuming](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment)
<details><summary>answer</summary>

```bash
k rollout pause deploy web
k set image deploy web nginx=nginx:1.19.9
k rollout resume deploy web
```
</details>

---

### 3.7 Autoscale `web` between 3 and 8 pods, CPU target 75% — [Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
<details><summary>answer</summary>

```bash
k autoscale deploy web --min=3 --max=8 --cpu=75%
k get hpa web
```

kubectl 1.34 and later take `--cpu=75%` (a percentage means average utilisation of the CPU
*request*; a quantity like `--cpu=500m` means average value). Older kubectl only has
`--cpu-percent=75`, which newer versions still accept with a deprecation warning. Check
`k autoscale -h` to see which one your kubectl has. Utilisation targets need `resources.requests.cpu`
on the pods and a running metrics-server, or the HPA shows `<unknown>` targets.
</details>

---

### 3.8 Write the YAML for a canary setup: 3 replicas of v1, 1 replica of v2, one Service selecting both via shared label `app: myapp` — [Canary deployments](https://kubernetes.io/docs/concepts/workloads/management/#canary-deployments)
<details><summary>answer</summary>

Service selects only `app: myapp` (shared label), both Deployments carry it.
Ratio is controlled by replica count (3:1 = 75%/25%).

```yaml
# service
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp       # matches both deployments
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# v1 deployment (3 replicas)
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
```

```yaml
# v2 deployment (1 replica)
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
```
</details>

---

### 3.9 Weighted traffic split across two Deployments: create `blue` (10 replicas, label `tier=web`) and `green` (10 replicas, label `tier=web`). Service `bg-svc` selects on `tier=web`. Scale to 70% blue, 30% green — [Deployment strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
<details><summary>answer</summary>

The shared `tier: web` label on both pod templates is what the Service selects. Traffic ratio is controlled by replica count.

```bash
k create deploy blue --image=nginx --replicas=10 --port=80 $do > blue.yaml
# add tier: web to spec.template.metadata.labels, apply
k create deploy green --image=nginx --replicas=10 --port=80 $do > green.yaml
# add tier: web to spec.template.metadata.labels, apply
k expose deploy blue --port=80 --name=bg-svc $do > svc.yaml   # blue must already exist
# change selector from app: blue to tier: web, apply
```

```yaml
# Service selector key — must match both deployments
selector:
  tier: web
```

```bash
# shift to 70/30
k scale deploy blue --replicas=7
k scale deploy green --replicas=3
```

The split is only approximate: kube-proxy spreads connections across all ready endpoints, so 7 of 10
pods gets roughly 70% of the traffic. A strict *blue/green* switch works differently: give each
Deployment its own label (`version: blue` / `version: green`) and change the Service's selector from
one to the other in a single step.
</details>

---

### 3.10 Change env var `TIER=web` to `TIER=app` on deployment `nginx-deployment` — [kubectl set env](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_env/)
<details><summary>answer</summary>

```bash
k set env deploy nginx-deployment TIER=app
k describe deploy nginx-deployment | grep -A1 -i env
```
</details>

---

### 3.11 Patch deployment `my-deployment` to set `revisionHistoryLimit: 20` using a strategic merge patch — [kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
<details><summary>answer</summary>

```bash
k patch deploy my-deployment -p '{"spec":{"revisionHistoryLimit":20}}'
k get deploy my-deployment -o jsonpath='{.spec.revisionHistoryLimit}'
```
</details>

---

### 3.12 Write the full YAML for a DaemonSet `node-logger` using `busybox` that runs `sleep 3600` on every node — [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
<details><summary>answer</summary>

There is no `kubectl create daemonset` — generate from a Deployment dry-run and change `kind`.

```bash
k create deploy node-logger --image=busybox $do -- sleep 3600 > ds.yaml
# edit: kind → DaemonSet, remove spec.replicas and spec.strategy
k apply -f ds.yaml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
      - name: busybox
        image: busybox
        args: ["sleep", "3600"]
```
</details>

---

### 3.13 What DaemonSet update strategy lets you update pods manually node-by-node? How do you set it? — [DaemonSet update](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)
<details><summary>answer</summary>

`OnDelete` — the pod is only replaced after you manually delete it on that node.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

Default is `RollingUpdate`. Use `OnDelete` when you need to control exactly when each node is updated.
</details>

---

### 3.14 Write the full YAML for a ReplicaSet `frontend`: 3 replicas of `nginx`, matching label `app=frontend` — [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
<details><summary>answer</summary>

There is no `kubectl create replicaset` — write it by hand or dry-run a Deployment and change `kind`.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
```
`spec.selector.matchLabels` **must** match `spec.template.metadata.labels`. If it doesn't, the API
server rejects the object with `` `selector` does not match template `labels` ``. Deployments and
DaemonSets are validated the same way. A ReplicaSet made from a Deployment dry-run already has
matching labels; just change `kind` and delete `strategy`.
</details>

---

### 3.15 Scale ReplicaSet `frontend` directly to 5 replicas, then delete one of its pods by hand — what happens? — [ReplicaSet — Non-Template Pod acquisitions](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#non-template-pod-acquisitions)
<details><summary>answer</summary>

```bash
k scale rs frontend --replicas=5
k delete pod <one-of-the-frontend-pods>
k get rs frontend    # DESIRED/CURRENT/READY still show 5 within seconds
```
The ReplicaSet controller notices the pod count dropped below `spec.replicas` and creates a
replacement immediately — this is the self-healing behavior a Deployment relies on its ReplicaSet
for. Also true of any pod matching `frontend`'s selector, even one you created by hand with a
matching label — the ReplicaSet will "adopt" it and count it toward `replicas`.
</details>

---

### 3.16 Check exactly what image revision 2 of Deployment `revtest` used, without diffing YAML by hand — [kubectl rollout history](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_history/)
<details><summary>answer</summary>

```bash
k rollout history deployment revtest --revision=2
```

Bare `k rollout history deployment revtest` only lists revision numbers with a `CHANGE-CAUSE`
column. That column is `<none>` unless someone set the `kubernetes.io/change-cause` annotation
(`k annotate deploy revtest kubernetes.io/change-cause="image 1.25"`); the old `--record` flag that
used to fill it in is deprecated. `--revision=N` prints that one revision's
full Pod template — image, env, resources — the actual detail you need to diagnose *what changed*
before deciding which revision to roll back to.
</details>

---

### 3.17 Deployment `web` reads a ConfigMap that you just changed. Recreate all its pods, one rolling step at a time, without editing the Deployment's YAML — [kubectl rollout restart](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/)
<details><summary>answer</summary>

```bash
k rollout restart deploy web
k rollout status deploy web
```

`rollout restart` adds a `kubectl.kubernetes.io/restartedAt` annotation to the pod template. That
changes the template, so the Deployment does a normal rolling update: it creates a new ReplicaSet,
adds a new revision to `rollout history`, and respects `maxSurge`/`maxUnavailable`. Use it when the
image hasn't changed but the pods need to pick up something new (ConfigMap or Secret values used as
env vars are only read when the container starts). It works on StatefulSets and DaemonSets too.
Deleting the pods by hand also gets them recreated, but all at once, with no rolling update.
</details>

---

## 4. Jobs & CronJobs

> **Docs:** [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) · [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

### 4.1 Create a job `pi` using `perl:5.34` that prints pi to 2000 digits — [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
<details><summary>answer</summary>

```bash
k create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(2000)'
```
</details>

---

### 4.2 Wait for job `pi` to complete, then get its logs — [kubectl wait](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_wait/)
<details><summary>answer</summary>

```bash
k wait --for=condition=complete --timeout=300s job pi
k logs job/pi
```
</details>

---

### 4.3 Create a job that runs busybox 5 times sequentially (completions) — [Job completions](https://kubernetes.io/docs/concepts/workloads/controllers/job/#completion-mode)
<details><summary>answer</summary>

```bash
k create job busybox --image=busybox $do -- /bin/sh -c 'echo hello' > job.yaml
# add to job.yaml:
#   spec.completions: 5
k apply -f job.yaml
```

```yaml
spec:
  completions: 5
```
</details>

---

### 4.4 Same job, but run 5 in parallel — [Job parallelism](https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs)
<details><summary>answer</summary>

```yaml
spec:
  parallelism: 5
```
</details>

---

### 4.5 Kill the job automatically if it takes more than 30 seconds — [Job deadline](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup)
<details><summary>answer</summary>

```yaml
spec:
  activeDeadlineSeconds: 30
```
</details>

---

### 4.6 Create a CronJob `heartbeat` that runs every minute printing the date — [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
<details><summary>answer</summary>

```bash
k create cronjob heartbeat --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c 'date'
```
</details>

---

### 4.7 CronJob should be skipped if it hasn't started within 17 seconds of its scheduled time. What field? — [startingDeadlineSeconds](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#starting-deadline)
<details><summary>answer</summary>

`spec.startingDeadlineSeconds: 17` on the CronJob.

```yaml
spec:
  startingDeadlineSeconds: 17
  schedule: "* * * * *"
```
</details>

---

### 4.8 Manually trigger a job from a CronJob named `heartbeat` — [kubectl create job](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_job/)
<details><summary>answer</summary>

```bash
k create job --from=cronjob/heartbeat heartbeat-manual
```
</details>

---

### 4.9 Bonus: three ways to delete completed Jobs — [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
<details><summary>answer</summary>

```bash
k get jobs -o name | xargs kubectl delete     # xargs can't see the `k` alias
k delete jobs --all
k delete jobs --field-selector=status.successful=1
```
`--field-selector=status.successful=1` is the one worth remembering: it deletes only Jobs that
have a successful pod and leaves failed and still-running ones alone. It matches the number exactly,
so it catches the usual single-completion Job; a Job with `completions: 5` finishes with
`status.successful=5`. Like `-l`, `--field-selector` works with `get` and `delete`.
</details>

---

### 4.10 Write a Job `pi` that gives up after 6 failed attempts, and is force-killed if still running after 10 seconds — [Job — Pod backoff failure policy](https://kubernetes.io/docs/concepts/workloads/controllers/job/#pod-backoff-failure-policy)
<details><summary>answer</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  backoffLimit: 6
  activeDeadlineSeconds: 10
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```
`backoffLimit: 6` is actually the **default** — writing it explicitly is a no-op unless you're
changing it. `activeDeadlineSeconds` is the harder deadline: once it elapses the whole Job is
terminated regardless of `backoffLimit`, and the Job's `status` reports reason `DeadlineExceeded`.
</details>

---

### 4.11 What does `completionMode: Indexed` change about a Job, versus the default? — [Job — Completion mode](https://kubernetes.io/docs/concepts/workloads/controllers/job/#completion-mode)
<details><summary>answer</summary>

Default is `NonIndexed`: the Job is done once **any** `.spec.completions` pods succeed — pods are
interchangeable, none of them know their position.

`Indexed`: each pod gets a completion index `0..completions-1`. The pod sees it as the env var
`JOB_COMPLETION_INDEX` and in the annotation `batch.kubernetes.io/job-completion-index`, and it's
part of the hostname (`$(job-name)-$(index)`) and pod name (`$(job-name)-$(index)-$(random)`).
The Job is done only once **every index** has one successful pod
— useful when each pod needs to work on a distinct shard/partition of a dataset rather than any pod
doing interchangeable work. Requires `completions` to be set; `parallelism` capped at 10^5.
```yaml
spec:
  completionMode: Indexed
  completions: 5
  parallelism: 5
```
</details>

---

### 4.12 CronJob `pi` should never run two instances at once — if the previous run hasn't finished when the next is due, skip it — [CronJob — Concurrency Policy](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#concurrency-policy)
<details><summary>answer</summary>

```yaml
spec:
  concurrencyPolicy: Forbid
```
Three values (`k explain cronjob.spec.concurrencyPolicy` lists them):
- `Allow` (**default**) — runs overlap freely.
- `Forbid` — skips the new run entirely if the previous one is still going.
- `Replace` — kills the still-running previous Job and starts the new one in its place.
</details>

---

### 4.13 Keep only the last 1 failed Job and last 3 successful Jobs in CronJob `pi`'s history — [CronJob — Jobs history limits](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#jobs-history-limits)
<details><summary>answer</summary>

```yaml
spec:
  failedJobsHistoryLimit: 1
  successfulJobsHistoryLimit: 3
```
These happen to be the **defaults** (1 and 3) — writing them explicitly only matters if you're
changing the retention count, e.g. `failedJobsHistoryLimit: 0` to keep zero failed Jobs around for
inspection.
</details>

---

### 4.14 Translate these cron schedules: `*/15 * * * *`, `0 2 * * *`, `0 9 * * 1-5` — [CronJob — Schedule syntax](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#schedule-syntax)
<details><summary>answer</summary>

```
# ┌───────────── minute (0–59)
# │ ┌───────────── hour (0–23)
# │ │ ┌───────────── day of month (1–31)
# │ │ │ ┌───────────── month (1–12)
# │ │ │ │ ┌───────────── day of week (0–6, Sunday=0 or 7; also sun/mon/tue/wed/thu/fri/sat)
# │ │ │ │ │
# * * * * *
```
- `*/15 * * * *` — every 15 minutes.
- `0 2 * * *` — 02:00 every day.
- `0 9 * * 1-5` — 09:00, Monday through Friday.

Know the field order (minute, hour, day of month, month, day of week) by heart. The CronJob page
linked in the heading has the same diagram, but it's quicker if you don't have to look it up.
</details>

---

## 5. Multi-container Pods

> **Docs:** [Pods — multiple containers](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers) · [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) · [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)

### 5.1 Write the YAML section to add a second container `busybox2` to an existing pod spec — [Multiple containers](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers)
<details><summary>answer</summary>

```yaml
containers:
- name: busybox
  image: busybox
  args: ["/bin/sh", "-c", "sleep 3600"]
- name: busybox2           # just copy and rename
  image: busybox
  args: ["/bin/sh", "-c", "sleep 3600"]
```
</details>

---

### 5.2 Exec into container `busybox2` inside pod `mypod` — [kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
<details><summary>answer</summary>

```bash
k exec -it mypod -c busybox2 -- /bin/sh
```
</details>

---

### 5.3 Write the full YAML for a pod with an init container that writes `"ready"` to `/shared/status`, and a main nginx container that mounts the same volume at `/usr/share/nginx/html` — [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
<details><summary>answer</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["/bin/sh", "-c", "echo ready > /shared/status"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared
    emptyDir: {}
```

Init containers run one at a time, in order, and each must exit successfully before the next one
starts. The app containers start only after all of them have finished. While one is running or
failing, the pod shows `Init:0/1` or `Init:CrashLoopBackOff`; read its output with
`k logs init-demo -c init`.
</details>

---

### 5.4 Two containers, `server` (nginx) and `client` (busybox), sit in the same Pod. How does `client` reach `server` — does it need a Service? — [Pods — networking](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking)
<details><summary>answer</summary>

```bash
k exec two-containers -c client -- wget -qO- http://localhost:80
```

No Service needed — **every container in a Pod always shares the same network namespace**, with
or without any special flag or field. One Pod = one IP = one loopback interface, so any container
in it reaches any other container in it via `localhost:<port>`, same as two processes on one
machine. This is *automatic* and unconditional — unlike `kubectl debug --target`, which shares the
*process* namespace and has to be requested explicitly. A Service is only needed to reach a
container in a *different* Pod.
</details>

---

### 5.5 What is a sidecar container? Write a Pod `web` where nginx writes its access log to a shared volume and a classic sidecar `log-tailer` (busybox) streams that log to stdout — [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
<details><summary>answer</summary>

A **sidecar** is a helper container that runs in the same Pod as the main app for the app's whole
life, adding something the app doesn't do itself: shipping logs, proxying traffic, syncing files,
reloading config. It shares the Pod's network (`localhost`) and whatever volumes you mount in both
containers. An *ambassador* (proxies outbound traffic for the app) and an *adapter* (converts the
app's output into a standard format) are specialised sidecars.

The **classic** way to add one is just a second entry under `containers:`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-tailer
    image: busybox
    command: ["sh", "-c", "tail -F /var/log/nginx/access.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

`k logs web -c log-tailer` then shows the access log. The weakness of a classic sidecar is that
Kubernetes treats it like any other app container. There's no guarantee it starts before the app,
and in a Job it keeps running after the main container exits, so the Job never completes. The
native sidecar in 5.6 fixes both.
</details>

---

### 5.6 Rewrite 5.5's `log-tailer` as a native sidecar. What makes it one, and what changes? — [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
<details><summary>answer</summary>

Move it to `initContainers` and give it `restartPolicy: Always`:

```yaml
spec:
  initContainers:
  - name: log-tailer
    image: busybox
    restartPolicy: Always          # this line makes it a native sidecar
    command: ["sh", "-c", "tail -F /var/log/nginx/access.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

- An init container with `restartPolicy: Always` doesn't have to exit before the next one starts.
  The kubelet starts it, waits until it's started (or its `startupProbe` passes), then carries on,
  so the sidecar is up before the app. It keeps running (and is restarted if it dies) for the
  whole life of the Pod.
- It doesn't keep a Job from completing: once the main containers finish, the sidecar is stopped.
  On shutdown it's stopped *after* the app containers.
- `restartPolicy: Always` is the only value allowed on a container, and only on init containers
  (`k explain pod.spec.initContainers.restartPolicy`).
- Native sidecars have been beta and on by default since Kubernetes 1.29, and GA since 1.33.
- `k logs web -c log-tailer` works the same way. The READY column counts the sidecar (`2/2`).
</details>

---

## 6. Configuration

> **Docs:** [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) · [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

### 6.1 Create a ConfigMap `app-config` with `ENV=prod` and `LOG_LEVEL=info` — [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
<details><summary>answer</summary>

```bash
k create cm app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=info
```
</details>

---

### 6.2 Create a ConfigMap from a file `config.txt`, key named `settings` — [Create ConfigMaps from files](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-the-key-to-use-when-creating-a-configmap-from-a-file)
<details><summary>answer</summary>

```bash
k create cm myconfig --from-file=settings=config.txt
```
</details>

---

### 6.3 Write the pod spec snippet to consume ConfigMap key `ENV` as env var `APP_ENV` — [ConfigMap as env vars](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-a-container-environment-variable-with-data-from-a-single-configmap)
<details><summary>answer</summary>

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: ENV
```
</details>

---

### 6.4 Write the pod spec snippet to mount ConfigMap `app-config` as a volume at `/etc/config` — [ConfigMap as volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap)
<details><summary>answer</summary>

```yaml
volumes:
- name: config-vol
  configMap:
    name: app-config
containers:
- name: nginx
  image: nginx
  volumeMounts:
  - name: config-vol
    mountPath: /etc/config
```
</details>

---

### 6.5 Create a Secret `db-creds` with `password=s3cr3t` — [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
<details><summary>answer</summary>

```bash
k create secret generic db-creds --from-literal=password=s3cr3t
```
</details>

---

### 6.6 Decode the value of key `password` from secret `db-creds` — [Decode a Secret](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/#decoding-secret)
<details><summary>answer</summary>

```bash
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```
</details>

---

### 6.7 Write the pod spec snippet to consume Secret key `password` as env var `DB_PASSWORD` — [Secret as env var](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#define-a-container-environment-variable-with-data-from-a-single-secret)
<details><summary>answer</summary>

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: password
```
</details>

---

### 6.8 Mount secret `db-creds` as a volume at `/etc/secrets` (read-only) — [Secret as volume](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#create-a-pod-that-has-access-to-the-secret-data-through-a-volume)
<details><summary>answer</summary>

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-creds
containers:
- name: nginx
  image: nginx
  volumeMounts:
  - name: secret-vol
    mountPath: /etc/secrets
    readOnly: true
```
</details>

---

### 6.9 Write the pod spec snippet to add Linux capabilities `NET_ADMIN` and `SYS_TIME` to a container — [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container)
<details><summary>answer</summary>

```yaml
containers:
- name: nginx
  image: nginx
  securityContext:
    capabilities:
      add: ["NET_ADMIN", "SYS_TIME"]
```
</details>

---

### 6.10 Create a LimitRange in namespace `dev` that sets max pod memory to 500Mi and min to 100Mi — [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
<details><summary>answer</summary>

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limits
  namespace: dev
spec:
  limits:
  - type: Pod
    max:
      memory: "500Mi"
    min:
      memory: "100Mi"
```
</details>

---

### 6.11 Create a ServiceAccount `deployer`, then assign it to an nginx pod — [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
<details><summary>answer</summary>

```bash
k create sa deployer
```

```yaml
spec:
  serviceAccountName: deployer
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 6.12 Write pod spec snippet to run container as user `1000` and enforce non-root — [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
<details><summary>answer</summary>

```yaml
containers:
- name: app
  image: nginx
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
```
</details>

---

### 6.13 Write pod spec snippet: read-only root filesystem, no privilege escalation — [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
<details><summary>answer</summary>

```yaml
containers:
- name: app
  image: nginx
  securityContext:
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
```
</details>

---

### 6.14 What is the difference between pod-level and container-level `securityContext`? Give an example of each — [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
<details><summary>answer</summary>

- `spec.securityContext` (pod-level): applies to all containers. Supports `runAsUser`, `runAsGroup`, `runAsNonRoot`, `fsGroup`, `supplementalGroups`, `sysctls`, `seccompProfile`
- `spec.containers[].securityContext` (container-level): overrides pod-level for that container. Only here: `capabilities`, `privileged`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation`

A common mistake is putting `capabilities` at pod level. `kubectl apply` rejects it with
`strict decoding error: unknown field "spec.securityContext.capabilities"`.

Container-level takes precedence over pod-level when both are set.

```yaml
spec:
  securityContext:           # pod-level: all containers run as user 1000
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:         # container-level: overrides/extends
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```
</details>

---

### 6.15 Generate an API token for service account `myuser` — [Configure Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount)
<details><summary>answer</summary>

```bash
k create token myuser
# longer-lived token:
k create token myuser --duration=8h
```

`k create token` prints a short-lived token (1 hour by default) and doesn't store it anywhere.
Since 1.24, creating a ServiceAccount no longer creates a token Secret for it automatically.
</details>

---

### 6.16 Create a pod that does NOT automount its service account token — [Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting)
<details><summary>answer</summary>

```yaml
spec:
  automountServiceAccountToken: false
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 6.17 Create a Secret for pulling images from a private registry given credentials directly (not a podman/docker auth file), and use it on a Pod — [Pull an image from a private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
<details><summary>answer</summary>

```bash
k create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user --docker-password=pass --docker-email=a@b.com
```

```yaml
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/app:v1
```

Distinct from 12.6, which builds the same `kubernetes.io/dockerconfigjson` secret type from an
*existing* podman/docker `auth.json` file (`--from-file=.dockerconfigjson=...`) — use
`docker-registry` when you have the raw username/password instead of an auth file already on disk.
Either way, the secret only takes effect once referenced via `spec.imagePullSecrets` on the Pod.
</details>

---

### 6.18 Create a ConfigMap from a `KEY=VALUE`-per-line file, one ConfigMap key per line — [ConfigMaps — from-env-file](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files)
<details><summary>answer</summary>

```bash
k create configmap app-config-env --from-env-file=env.txt
```

Different behavior from `--from-file` (6.2): `--from-file=settings=config.txt` stores the *whole
file* as one blob under key `settings`. `--from-env-file` parses each `KEY=VALUE` line into its
*own* ConfigMap key (blank lines and `#`-comments ignored) — the shape you want when the file is
meant to become individual env vars, not one opaque file.
</details>

---

## 7. Observability

> **Docs:** [Liveness, Readiness & Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) · [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)

### 7.1 Write a liveness probe that runs `ls` every 5 seconds, starting after 5 seconds — [Exec liveness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command)
<details><summary>answer</summary>

```yaml
livenessProbe:
  exec:
    command:
    - ls
  initialDelaySeconds: 5
  periodSeconds: 5
```
</details>

---

### 7.2 Write an HTTP readiness probe on path `/healthz` port `8080` — [HTTP readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)
<details><summary>answer</summary>

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```
</details>

---

### 7.3 Write a TCP liveness probe on port `3306` — [TCP liveness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe)
<details><summary>answer</summary>

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```
</details>

---

### 7.4 Pod `app` crashed on startup — no logs show. What do you do? — [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
<details><summary>answer</summary>

```bash
k describe po app          # check Events section
k get events | grep app    # broader event search
k logs app -p              # logs from previous container instance
```
</details>

---

### 7.5 Find pods with liveness probe failures in the current namespace — [kubectl get events](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get events --field-selector reason=Unhealthy
```

To narrow to liveness only (vs readiness):

```bash
k get events --field-selector reason=Unhealthy | grep Liveness
```
</details>

---

### 7.6 List all pods cluster-wide sorted by CPU, then separately sorted by memory — [kubectl top](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/)
<details><summary>answer</summary>

```bash
k top pods -A --sort-by=cpu
k top pods -A --sort-by=memory
```
</details>

---

### 7.7 Get all events across all namespaces sorted by creation timestamp — [kubectl get events](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get events -A --sort-by=.metadata.creationTimestamp
```
</details>

---

### 7.8 Get logs from pod `log-pod` for the last hour — [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
<details><summary>answer</summary>

```bash
k logs log-pod --since=1h
```
</details>

---

### 7.9 Convert an Ingress manifest from `networking.k8s.io/v1beta1` to `networking.k8s.io/v1` — [Migrate to non-deprecated APIs](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
<details><summary>answer</summary>

```bash
kubectl-convert -f old-ingress.yaml --output-version networking.k8s.io/v1
# pipe straight into apply if you want to update the cluster
kubectl-convert -f old-ingress.yaml --output-version networking.k8s.io/v1 | k apply -f -
```

Requires the `kubectl-convert` plugin, which is installed separately and may not be available.
Without it, convert by hand. For Ingress, the v1 changes are: `backend.serviceName`/`servicePort`
become `backend.service.name` and `backend.service.port.number` (or `.name`), every path needs a
`pathType`, and the `kubernetes.io/ingress.class` annotation becomes `spec.ingressClassName`.
`k explain ingress.spec.rules.http.paths --api-version=networking.k8s.io/v1` shows the new shape.
</details>

---

### 7.10 Find all events related to Deployment `web` in namespace `dev`, oldest first — [kubectl get events](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get events -n dev --field-selector involvedObject.name=web --sort-by=.metadata.creationTimestamp
```

`k explain event.involvedObject` shows the field exists, but it won't tell you which fields
`--field-selector` accepts. That list is different for each resource and you have to learn it. For
Events it's `involvedObject.kind/name/namespace/uid`, `reason`, `type` and `source`. An unsupported
field fails with `field label not supported`. This only matches events *about the Deployment
itself* (e.g. `ScalingReplicaSet`). Events for its pods are recorded under each pod's name. See
[guide/exam-tips.md](../guide/exam-tips.md) for more event-filtering patterns.
</details>

---

### 7.11 Delete a pod that's stuck `Terminating` and won't go away — [Force delete pods](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)
<details><summary>answer</summary>

```bash
k delete pod stuck-pod $now        # $now = --force --grace-period=0
```

A plain `k delete pod` waits out the container's graceful-shutdown period (30s by default) and can
hang indefinitely if the node or kubelet is unreachable. `--force --grace-period=0` removes the
object from the API straight away without waiting for the kubelet to confirm the container stopped.
In the exam it's also the quick way to delete any pod you're about to recreate (e.g. after editing a
field that can't be changed on a running pod), not just stuck ones. kubectl prints a warning that
the container may still be running on the node; that's expected.
</details>

---

### 7.12 A legacy app can take up to 5 minutes to start, and its liveness probe (`httpGet /healthz:8080`, every 10s) keeps killing it during startup. Fix it without making the liveness probe slower — [Startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes)
<details><summary>answer</summary>

```yaml
containers:
- name: app
  image: legacy-app
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30       # 30 × 10s = up to 300s to start
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    periodSeconds: 10
```

While a `startupProbe` is defined and hasn't succeeded yet, the liveness and readiness probes don't
run. The time the app gets to start is `failureThreshold × periodSeconds`, plus
`initialDelaySeconds` if you set it. Here that's 300s. If the startup probe hasn't passed by then,
the container is killed and restarted as usual. After its first success it never runs again, and the
normal liveness probe takes over at its own fast interval. Raising `initialDelaySeconds` on the
liveness probe would also get the app through startup, but then a hung app wouldn't be restarted
for 5 minutes either.
</details>

---

### 7.13 Pod `api` runs a distroless image with no shell. Attach a busybox shell that can see the `api` container's processes, without restarting the pod — [Ephemeral containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)
<details><summary>answer</summary>

```bash
k debug -it api --image=busybox --target=api
# inside: ps aux  → the app's processes are visible
```

`k debug <pod>` adds an **ephemeral container** to the running pod. It shows up under
`spec.ephemeralContainers` and the pod isn't restarted. `--target=<container>` puts the debug
container in that container's process namespace, so `ps` shows the app's processes and
`/proc/<pid>/root` shows its filesystem. Without `--target` you still share the pod's network
(`localhost`), but you can't see the app's processes.
Ephemeral containers can't be removed once added; they stay in the pod spec (stopped) until the
pod is deleted.
</details>

---

### 7.14 Pod `api` is in `CrashLoopBackOff`, so there's nothing running to attach to. Debug it on a copy of the pod instead, with a busybox container added and the process namespace shared — [Debugging using a copy of the Pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#debugging-using-a-copy-of-the-pod)
<details><summary>answer</summary>

```bash
k debug api -it --image=busybox --copy-to=api-debug --share-processes
# afterwards
k delete pod api-debug $now
```

`--copy-to=<new-name>` creates a **new pod** that's a copy of `api` plus the debug container, and
leaves the original alone. `--share-processes` sets `shareProcessNamespace: true` on the copy so
the debug container can see the app's processes (it's on by default with `--copy-to`, but saying
it makes the intent clear). Related flags that only work together with `--copy-to`:

```bash
# same copy, but swap the crashing container's image
k debug api --copy-to=api-debug --set-image=api=busybox
# same copy, but replace the command of container api with a shell to poke around
k debug api -it --copy-to=api-debug --container=api -- sh
```

Remember to delete the copy; it isn't managed by any controller.
</details>

---

## 8. Services & Networking

> **Docs:** [Services](https://kubernetes.io/docs/concepts/services-networking/service/) · [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

### 8.1 Create an nginx pod on port 80, and expose it as a ClusterIP service in one command — [kubectl run --expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run nginx --image=nginx --port=80 --expose --restart=Never
```

Creates `pod/nginx` and a ClusterIP `service/nginx` whose selector is the pod's `run=nginx` label.
`--expose` needs `--port`.
</details>

---

### 8.2 Expose deployment `web` on service port 8080, target port 3000 — [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
<details><summary>answer</summary>

```bash
k expose deploy web --port=8080 --target-port=3000
```
</details>

---

### 8.3 Convert service `nginx` from ClusterIP to NodePort imperatively — [Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
<details><summary>answer</summary>

```bash
k patch svc nginx -p '{"spec":{"type":"NodePort"}}'
```
</details>

---

### 8.4 Write a NetworkPolicy that allows only pods with label `role=frontend` to access pods with label `app=db` on port 5432 — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - port: 5432
```
</details>

---

### 8.5 Test connectivity: hit service `mysvc` on port 80 from a temporary busybox pod — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run test --image=busybox --rm -it --restart=Never -- wget -qO- -T 2 mysvc:80
```

`-T 2` (or `--timeout 2`) makes busybox `wget` give up after 2 seconds instead of hanging when a
NetworkPolicy drops the traffic. From a different namespace, use `mysvc.<namespace>` or the full
`mysvc.<namespace>.svc.cluster.local`.
</details>

---

### 8.6 Write an Ingress `my-ingress` routing all traffic at path `/` to service `my-service` on port 8080 — [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 8080
```

Faster: `k create ingress my-ingress --class=nginx --rule="/*=my-service:8080" $do` generates the
same thing (the trailing `*` gives `pathType: Prefix`). Use the IngressClass that exists in the
cluster (`k get ingressclass`). A `nginx.ingress.kubernetes.io/rewrite-target` annotation is only
needed when the backend expects a different path from the one in the URL.
</details>

---

### 8.7 Write a NetworkPolicy that denies ALL ingress traffic to pods with label `app=web` — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress: []
```

An empty `ingress` list with `policyTypes: [Ingress]` blocks all inbound traffic.
</details>

---

### 8.8 Write a NetworkPolicy allowing ingress to `app=api` on port 80 only from pods in namespaces labeled `env=prod` — [namespaceSelector](https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-prod-ns
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: prod
    ports:
    - port: 80
```
</details>

---

### 8.9 Write a NetworkPolicy that denies ALL egress from pods with label `app=restricted` — [Egress rules](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-restricted
spec:
  podSelector:
    matchLabels:
      app: restricted
  policyTypes:
  - Egress
  egress: []
```
</details>

---

### 8.10 Write two NetworkPolicies: `tier=web` pods can reach `tier=app` pods on port 80 (egress), and `tier=app` pods only accept from `tier=web` on port 80 (ingress) — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-egress-to-app
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: app
    ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-ingress-from-web
spec:
  podSelector:
    matchLabels:
      tier: app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - port: 80
```
</details>

---

### 8.11 Create an Ingress `simple` imperatively: host `foo.com`, path `/bar`, routing to service `svc1:8080` — [kubectl create ingress](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_ingress/)
<details><summary>answer</summary>

```bash
k create ingress simple --rule="foo.com/bar=svc1:8080"
```

`--rule` syntax: `host/path=service:port[,tls[=secretname]]`. Repeat `--rule` for multiple paths/hosts.
</details>

---

### 8.12 Create an Ingress `catch-all` with no host restriction, path `/path`, IngressClass `otherclass` — [kubectl create ingress](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_ingress/)
<details><summary>answer</summary>

```bash
k create ingress catch-all --class=otherclass --rule="/path=svc:port"
```

Omitting the host before the `/` matches all hosts. The part after `:` can be a port number
(`svc:8080` → `port.number`) or a named Service port (`svc:http` → `port.name`).
</details>

---

### 8.13 Create an Ingress `multipath` on host `foo.com` with two paths: `/` → `svc:port`, `/admin/` → `svcadmin:portadmin` — [kubectl create ingress](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_ingress/)
<details><summary>answer</summary>

```bash
k create ingress multipath --class=default \
  --rule="foo.com/=svc:port" \
  --rule="foo.com/admin/=svcadmin:portadmin"
```

One `--rule` per path — they stack into `spec.rules[].http.paths[]` under the same host.
Without a trailing `*` both paths get `pathType: Exact`, so `foo.com/` only matches `/` itself, not
`/anything`. Write `foo.com/*=svc:port` for a prefix match.
</details>

---

### 8.14 Create an Ingress `ingtls` for host `foo.com` path `/`, backend `svc:port`, with TLS terminated using existing secret `my-cert` — [Ingress TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
<details><summary>answer</summary>

```bash
k create ingress ingtls --class=default --rule="foo.com/=svc:port,tls=my-cert"
```

This adds `spec.tls: [{hosts: [foo.com], secretName: my-cert}]`. The Secret must be of type
`kubernetes.io/tls` in the same namespace (`k create secret tls my-cert --cert=... --key=...`).
`,tls` with no `=secret` uses the ingress controller's default certificate instead of a named Secret.
</details>

---

### 8.15 Generate the YAML for an Ingress (don't create it) routing `foo.com/*` to `svc:8080`, with a default backend of `defaultsvc:80` — [kubectl create ingress](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_ingress/)
<details><summary>answer</summary>

```bash
k create ingress withdefault --class=default \
  --default-backend=defaultsvc:80 \
  --rule="foo.com/*=svc:8080" $do
```

Trailing `*` on the path sets `pathType: Prefix` instead of the default `Exact`.
`--default-backend` sets `spec.defaultBackend`, used when no rule matches.
</details>

---

### 8.16 A Deployment `web` in namespace `shop` has no Service. From another pod, what must exist before `http://web` resolves, and how do you verify it? — [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
<details><summary>answer</summary>

```bash
# a bare Deployment/Pod name never gets a DNS record — only a Service does
k expose deploy web -n shop --port=80 --target-port=80

# verify from a temporary pod
k run tmp --rm -it --restart=Never -n shop --image=curlimages/curl -- curl -m 5 http://web
```

The short name `web` only resolves from pods in the same namespace. From anywhere else use
`web.shop` or `web.shop.svc.cluster.local`.
</details>

---

### 8.17 Write an Egress NetworkPolicy for `tier=app` pods: allow egress to `tier=db` pods on port 5432, and allow DNS resolution to anywhere — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-egress-db-and-dns
spec:
  podSelector:
    matchLabels:
      tier: app
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: db
    ports:
    - port: 5432
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

Two separate top-level rules under `egress:` — DNS must be its own rule with no `to:`, otherwise
it's scoped to only the `db` pods instead of allowed to anywhere (CoreDNS lives in `kube-system`).
Forgetting this rule breaks all name resolution for the pod, not just the traffic being restricted.
</details>

---

### 8.18 Write a NetworkPolicy allowing ALL ingress traffic to `app=web` pods (the opposite of 8.7's deny-all) — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-all
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - {}
```

One character different from 8.7's deny-all (`ingress: []`), opposite meaning: `ingress: - {}` is
one rule with no restrictions at all (allow everything), `ingress: []` is zero rules (allow
nothing). Easy to transpose under pressure — read it as "how many rules" (`[]` = none) vs. "how
restricted is the one rule" (`{}` = not at all).
</details>

---

### 8.19 Write a NetworkPolicy allowing ingress to `app=db` only from pods labeled `role=frontend` **in namespaces labeled `env=prod`** (both conditions on the same source) — [Network Policies — from and to selectors](https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-prod-frontend-and
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: prod
      podSelector:
        matchLabels:
          role: frontend
```

`namespaceSelector` and `podSelector` as two keys on the *same* `from:` list item is an AND — the
source pod must match both. List them as two separate `-` items instead (compare 8.8) and it
becomes an OR — either condition alone is enough. Same YAML-nesting-changes-meaning trap as 8.17's
egress rules, one level further in.
</details>

---

### 8.20 Write a NetworkPolicy allowing ingress to `app=db` from pods with `role=frontend`, but only within `db`'s own namespace (no cross-namespace access) — [Network Policies — from and to selectors](https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-frontend-same-ns
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

Omitting `namespaceSelector` entirely (just a bare `podSelector` under `from:`) restricts matching
to the **same namespace** as the policy — that's the default scope, not "all namespaces." Contrast
with an explicit `namespaceSelector: {}` (no `matchLabels`), which means the opposite: match pods
in *every* namespace. The two look like minor variations of the same idea; they're opposite in
scope.
</details>

---

### 8.21 A Service remaps port 8001 → container port 8000. What port does a NetworkPolicy restricting traffic to that Service's pods need to reference? — [Network Policies — ports](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
ingress:
- from: [...]
  ports:
  - port: 8000   # the pod's containerPort, NOT the Service's 8001
```

`NetworkPolicy` operates at the pod/traffic level and has no concept of a Service's port mapping —
`ports:` always means the destination pod's actual `containerPort`, never a Service's exposed
port. A named port is a less error-prone alternative: `ports: - port: api-port` in the policy,
matching `ports: - name: api-port, containerPort: 8000` on the container, so the policy doesn't
silently go stale if the container port number ever changes.
</details>

---

### 8.22 Write a NetworkPolicy allowing ingress on port 80 to `app=web` pods from absolutely anywhere — internal pods and external LoadBalancer traffic alike — [Network Policies — ports without from](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-port-80-from-anywhere
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - port: 80
```

A `ports:`-only rule with no `from:` at all restricts *which port* is open without restricting *who*
can reach it — useful for narrowing an otherwise wide-open `ingress: - {}` (8.18) down to just the
port the app actually serves, while still allowing both in-cluster pods and an external
LoadBalancer/NodePort's traffic to reach it.
</details>

---

### 8.23 A Service "exists" but a client gets connection refused / times out reaching it — what's the standard first command to check, before touching DNS or the pods? — [Debug Services — endpoints](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/#does-the-service-have-any-endpointslices)
<details><summary>answer</summary>

```bash
k get endpoints my-svc
# or across the namespace:
k get ep
# the newer API (kubectl warns that v1 Endpoints is deprecated from 1.33 on):
k get endpointslices -l kubernetes.io/service-name=my-svc
```

This shows which pod IPs the Service is actually sending traffic to right now. Empty `ENDPOINTS`
means no *ready* pod matches the Service's selector. Either the selector doesn't match the pod
labels (compare `k get svc my-svc -o wide` with `k get po --show-labels`), or the pods match but
are failing their readiness probe. It's the quickest way to tell "Service misconfigured" apart from
"DNS broken" before going further. Also check that the Service's `targetPort` is the port the
container really listens on.
</details>

---

### 8.24 A NetworkPolicy already restricts ingress to `app=db` pods to only those labeled `role=frontend`. Pod `client` (no such label) can't connect, and the policy must not be touched — fix it — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
<details><summary>answer</summary>

```bash
k get networkpolicy -n <ns> -o yaml    # read the existing podSelector under `from` — confirm the
                                        # exact label key/value it requires, don't guess
k label pod client role=frontend -n <ns>
```
This is more common on the exam than writing a NetworkPolicy from scratch: the policy is already
correct and immutable, and the fix is only ever a missing/wrong label on the client pod.
</details>

---

## 9. State / Persistence

> **Docs:** [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### 9.1 Write the full YAML for a PersistentVolume `mypv`: 5Gi, ReadWriteOnce, hostPath `/data`, storageClass `manual` — [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)
<details><summary>answer</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /data
```
</details>

---

### 9.2 Write the PVC that binds to the PV above: request 2Gi — [PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
<details><summary>answer</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 2Gi
```
</details>

---

### 9.3 Write the pod spec snippet to mount PVC `mypvc` at `/data` — [Persistent Volumes in Pods](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)
<details><summary>answer</summary>

```yaml
volumes:
- name: data-vol
  persistentVolumeClaim:
    claimName: mypvc
containers:
- name: app
  image: busybox
  volumeMounts:
  - name: data-vol
    mountPath: /data
```
</details>

---

### 9.4 Two pods both mount the same PVC using `ReadWriteOnce`. Pod 2 can't see Pod 1's files on a multi-node cluster. Why? Fix? — [Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
<details><summary>answer</summary>

`hostPath` PVs are node-local. If pods land on different nodes, each one sees its own node's
directory, so they don't share files.

`ReadWriteOnce` means the volume can be mounted read-write by **one node** at a time, not one pod.
Pods on the same node can share it. With real network block storage (a cloud disk), a second pod on
another node doesn't silently get an empty directory: it gets stuck in `ContainerCreating` with a
`Multi-Attach error` event. `ReadWriteOncePod` is the mode that limits a volume to a single pod.

Fix: use storage that supports `ReadWriteMany` (NFS, CephFS, a cloud file service) and request that
access mode, or make sure both pods land on the same node (node affinity, or a `nodeSelector` on the
same node).
</details>

---

## 10. Helm

> **Docs:** [Helm Docs](https://helm.sh/docs/) · [Helm CLI reference](https://helm.sh/docs/helm/)

### 10.1 Add the Bitnami repo and install `bitnami/nginx` as release `web` with 3 replicas — [helm install](https://helm.sh/docs/helm/helm_install/)
<details><summary>answer</summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm show values bitnami/nginx | grep -i replica   # find the key name
helm install web bitnami/nginx --set replicaCount=3
```
</details>

---

### 10.2 Upgrade release `web` using a custom values file — [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
<details><summary>answer</summary>

```bash
helm upgrade web bitnami/nginx -f my-values.yaml
```
</details>

---

### 10.3 List all Helm releases that are in a pending state across all namespaces — [helm list](https://helm.sh/docs/helm/helm_list/)
<details><summary>answer</summary>

```bash
helm list --pending -A
```
</details>

---

### 10.4 Uninstall release `web` from namespace `prod` — [helm uninstall](https://helm.sh/docs/helm/helm_uninstall/)
<details><summary>answer</summary>

```bash
helm uninstall web -n prod
```
</details>

---

### 10.5 Download (not install) the `bitnami/redis` chart and untar it — [helm pull](https://helm.sh/docs/helm/helm_pull/)
<details><summary>answer</summary>

```bash
helm pull bitnami/redis --untar
```
</details>

---

### 10.6 Show the revision history of release `web`, then roll it back to revision 2 — [helm rollback](https://helm.sh/docs/helm/helm_rollback/)
<details><summary>answer</summary>

```bash
helm history web
helm rollback web 2
helm history web      # a new revision appears: "Rollback to 2"
```

A rollback doesn't delete later revisions. It creates a **new** revision (e.g. 4) with revision 2's
chart and values, just as `kubectl rollout undo` does for a Deployment. `helm rollback web` with no
revision number goes back to the previous one. Add `-n <namespace>` if the release isn't in your
current namespace: `helm history`/`rollback` look only there, and you get `release: not found`.
</details>

---

### 10.7 Show the values you set when installing release `web`, then every value the chart is actually using — [helm get values](https://helm.sh/docs/helm/helm_get_values/)
<details><summary>answer</summary>

```bash
helm get values web              # only USER-SUPPLIED VALUES (your --set / -f)
helm get values web --all        # COMPUTED VALUES: chart defaults merged with yours
helm get values web --revision 2 # what was set at an older revision
```

`helm get values` answers "what did someone change on this release?". `helm show values <chart>`
shows the chart's defaults before anything is installed. Other `helm get` subcommands:
`helm get manifest web` prints the rendered Kubernetes YAML, and `helm get all web` prints everything.
</details>

---

### 10.8 Release `web` was installed with `--set replicaCount=3`. Upgrade it to set `image.tag=1.27` without losing the replica count — [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
<details><summary>answer</summary>

```bash
helm upgrade web bitnami/nginx --reuse-values --set image.tag=1.27
helm get values web   # replicaCount: 3 and image.tag: "1.27"
```

The trap: if you pass any `--set` or `-f` to `helm upgrade`, Helm starts again from the chart's
defaults and applies only what you passed this time. `helm upgrade web bitnami/nginx --set
image.tag=1.27` would quietly put `replicaCount` back to the chart default. `--reuse-values` takes
the previous release's values and adds your new `--set` on top. The other way to do it is to repeat
every override (`--set replicaCount=3 --set image.tag=1.27`) or keep them all in a values file.
`helm upgrade --install` installs the release if it doesn't exist yet.
</details>

---

### 10.9 Find what chart versions of `bitnami/nginx` are available, then upgrade release `web` to version `15.5.7` specifically — [helm search repo](https://helm.sh/docs/helm/helm_search_repo/)
<details><summary>answer</summary>

```bash
helm search repo bitnami/nginx --versions   # -l is the short flag; lists every cached version
helm upgrade web bitnami/nginx --version 15.5.7
```
`helm show chart bitnami/nginx` is the other way to check a single already-known chart's
`version`/`appVersion` without listing the whole version history. Without `--version`, `helm
upgrade` always takes the latest.
</details>

---

## 11. CRD

> **Docs:** [Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

### 11.1 Write the CRD manifest for a `Widget` resource in group `acme.io`, namespaced, with fields `color: string` and `size: integer` — [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
<details><summary>answer</summary>

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.acme.io
spec:
  group: acme.io
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
    shortNames:
    - wg
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              color:
                type: string
              size:
                type: integer
```
</details>

---

### 11.2 Create a custom object of kind `Widget` named `my-widget` with `color: blue`, `size: 3` — [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
<details><summary>answer</summary>

```yaml
apiVersion: acme.io/v1
kind: Widget
metadata:
  name: my-widget
spec:
  color: blue
  size: 3
```
</details>

---

### 11.3 List all widgets using the short name — [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get wg
```
</details>

---

### 11.4 What is an Operator? An operator was installed in your cluster — find which CRDs it added and what kinds they define, then create one of its custom resources — [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
<details><summary>answer</summary>

An **Operator** is a controller (usually a Deployment running in the cluster) plus one or more CRDs.
You describe what you want in a custom resource (e.g. `kind: Database`, `spec.replicas: 3`), and
the operator's controller watches those resources and creates or fixes the underlying Pods,
Services, Secrets, etc. until the cluster matches. The CRD only adds the new API type. Without the
controller, a custom resource is just stored data and nothing acts on it.

```bash
k get crd                                   # all CRDs, named <plural>.<group>
k get crd | grep acme                       # narrow to the operator's API group
k api-resources --api-group=acme.io         # kinds, short names, namespaced?, API version
k explain widget.spec                       # fields the CR accepts (from the CRD's schema)
k get widgets -A                            # existing instances
```

Then write a CR using the `APIVERSION` and `KIND` columns from `api-resources`:

```yaml
apiVersion: acme.io/v1
kind: Widget
metadata:
  name: demo
spec:
  color: red
  size: 1
```

`k apply -f widget.yaml`, then `k get widgets` (or the short name). Right after a CRD is created,
`explain` and short names can take a few seconds to show up while kubectl refreshes its discovery
cache. To see what the operator did
with it, check the CR's `status` (`k describe widget demo`) and the operator's own logs
(`k logs deploy/<operator> -n <operator-namespace>`).
</details>

---

## 12. Podman

> **Docs:** [Podman docs](https://docs.podman.io/en/latest/) · [Podman CLI reference](https://docs.podman.io/en/latest/Commands.html)

### 12.1 Build image `myapp` from the current directory — [podman build](https://docs.podman.io/en/latest/markdown/podman-build.1.html)
<details><summary>answer</summary>

```bash
podman build -t myapp .
```
</details>

---

### 12.2 Run `myapp` as a container named `test`, mapping host port 8080 to container port 80, detached — [podman run](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
<details><summary>answer</summary>

```bash
podman run -d --name test -p 8080:80 myapp
```
</details>

---

### 12.3 Show the image layers of `myapp` — [podman image tree](https://docs.podman.io/en/latest/markdown/podman-image-tree.1.html)
<details><summary>answer</summary>

```bash
podman image tree myapp
```
</details>

---

### 12.4 Export container `test` to `backup.tar` — [podman export](https://docs.podman.io/en/latest/markdown/podman-export.1.html)
<details><summary>answer</summary>

```bash
podman export test --output=backup.tar
```
</details>

---

### 12.5 Tag `myapp` for a local registry at `localhost:5000` and push it — [podman push](https://docs.podman.io/en/latest/markdown/podman-push.1.html)
<details><summary>answer</summary>

```bash
podman tag myapp localhost:5000/myapp
podman push localhost:5000/myapp
```
</details>

---

### 12.6 Create a `kubernetes.io/dockerconfigjson` secret from existing podman login credentials — [Pull an image from a private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
<details><summary>answer</summary>

```bash
# after: podman login docker.io
k create secret generic regcred \
  --from-file=.dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json \
  --type=kubernetes.io/dockerconfigjson
```
</details>

---

### 12.7 Build `myapp:v2` from the Dockerfile in the current directory, tag it, and save it to `/opt/backups/myapp.tar` in OCI format — [podman save](https://docs.podman.io/en/latest/markdown/podman-save.1.html)
<details><summary>answer</summary>

```bash
podman build -t myapp:v2 .
podman save --format oci-archive -o /opt/backups/myapp.tar myapp:v2
```
`--format` defaults to `docker-archive` — it must be set explicitly to `oci-archive` for OCI. Don't
confuse this with `podman export` (12.4), which dumps a *running/stopped container's* filesystem as
a flat tar with no image manifest/layers — not restorable with `podman load`.
</details>

---

## 13. yq — In-place YAML editing (optional, not in the curriculum)

> **Optional.** yq isn't part of the CKAD curriculum, may not be installed in the exam environment,
> and its docs aren't on the list of pages you may open. Skip this section if you're short on time;
> in the exam, edit YAML with vim or `kubectl edit`/`kubectl patch`.
>
> **Docs:** [yq](https://mikefarah.gitbook.io/yq/) (yq v4 by Mike Farah)
> Pattern: generate YAML with `$do`, save it to a file, edit it with `yq`, then `k apply`.

### 13.1 Change the replica count in `deploy.yaml` to 5 — [yq assign](https://mikefarah.gitbook.io/yq/operators/assign-update)
<details><summary>answer</summary>

```bash
yq e '.spec.replicas = 5' -i deploy.yaml
```
</details>

---

### 13.2 Change the image of the first container in `pod.yaml` to `nginx:1.24` — [yq traverse](https://mikefarah.gitbook.io/yq/operators/traverse-read)
<details><summary>answer</summary>

```bash
yq e '.spec.containers[0].image = "nginx:1.24"' -i pod.yaml
```
</details>

---

### 13.3 Add label `env=prod` to the pod template in `deploy.yaml` — [yq assign](https://mikefarah.gitbook.io/yq/operators/assign-update)
<details><summary>answer</summary>

```bash
yq e '.spec.template.metadata.labels.env = "prod"' -i deploy.yaml
```
</details>

---

### 13.4 Set `activeDeadlineSeconds: 30` on `job.yaml` — [yq assign](https://mikefarah.gitbook.io/yq/operators/assign-update)
<details><summary>answer</summary>

```bash
yq e '.spec.activeDeadlineSeconds = 30' -i job.yaml
```
</details>

---

### 13.5 Read the current image of the first container from `deploy.yaml` without opening the file — [yq traverse](https://mikefarah.gitbook.io/yq/operators/traverse-read)
<details><summary>answer</summary>

```bash
yq e '.spec.template.spec.containers[0].image' deploy.yaml
```
</details>

---

### 13.6 Full workflow: generate a Job YAML, set completions to 5, then apply — [yq](https://mikefarah.gitbook.io/yq/)
<details><summary>answer</summary>

```bash
k create job myjob --image=busybox $do -- /bin/sh -c 'echo hi' > job.yaml
yq e '.spec.completions = 5' -i job.yaml
k apply -f job.yaml
```
</details>

---

## 14. Cluster Introspection

> **Docs:** [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

### 14.1 Show the currently active namespace — [kubectl config get-contexts](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_get-contexts/)
<details><summary>answer</summary>

```bash
k config get-contexts
# The * row is the active context; the NAMESPACE column shows the active namespace
# (empty means "default")
k config view --minify -o jsonpath='{..namespace}'   # just the namespace
```
</details>

---

### 14.2 List all resource types that exist inside a namespace (names only) — [kubectl api-resources](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_api-resources/)
<details><summary>answer</summary>

```bash
k api-resources --namespaced=true -o name
```
</details>

---

### 14.3 Set memory request 25Mi and limit 100Mi on all containers in deployment `my-deployment` — [Manage resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
<details><summary>answer</summary>

```bash
k set resources deploy my-deployment --requests=memory=25Mi --limits=memory=100Mi
k describe deploy my-deployment | grep -A4 Limits
```
</details>

---

### 14.4 Switch the active namespace to `staging` without changing the context name — [kubectl config set-context](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_set-context/)
<details><summary>answer</summary>

```bash
k config set-context --current --namespace=staging
k config get-contexts   # verify NAMESPACE column
```

Useful when a task does all its work in one namespace, so you don't have to add `-n staging` to
every command. In the exam, each task tells you which context or host to use first. Run the command
it gives you, and don't assume the namespace from the previous task still applies. Passing `-n`
explicitly on every command is the safer habit.
</details>

---

### 14.5 Show only the name of the currently active context — [kubectl config current-context](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_current-context/)
<details><summary>answer</summary>

```bash
k config current-context
```
</details>

---

### 14.6 View the full merged kubeconfig, then a minified view of only the active context — [kubectl config view](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_view/)
<details><summary>answer</summary>

```bash
k config view
k config view --minify   # only the active context/cluster/user
```
</details>

---

## 15. RBAC

> **Docs:** [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

### 15.1 Create a Role `pod-reader` in namespace `dev` that allows `get`, `list`, `watch` on pods — [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-example)
<details><summary>answer</summary>

```bash
k create role pod-reader --verb=get,list,watch --resource=pods -n dev
```
</details>

---

### 15.2 Bind Role `pod-reader` to user `alice` in namespace `dev` — [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-example)
<details><summary>answer</summary>

```bash
k create rolebinding pod-reader-alice --role=pod-reader --user=alice -n dev
```
</details>

---

### 15.3 Create a ClusterRole `node-reader` that allows `get`, `list` on nodes — [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#clusterrole-example)
<details><summary>answer</summary>

```bash
k create clusterrole node-reader --verb=get,list --resource=nodes
```
</details>

---

### 15.4 Bind ClusterRole `node-reader` to ServiceAccount `myuser` in namespace `dev` — [ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#clusterrolebinding-example)
<details><summary>answer</summary>

```bash
k create clusterrolebinding node-reader-myuser \
  --clusterrole=node-reader \
  --serviceaccount=dev:myuser
```

Note: `--serviceaccount` takes `namespace:name` format.
</details>

---

### 15.5 Check if user `alice` can list pods in namespace `dev` — [kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
<details><summary>answer</summary>

```bash
k auth can-i list pods --as=alice -n dev
```
</details>

---

### 15.6 Check if ServiceAccount `myuser` in namespace `dev` can create deployments — [kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
<details><summary>answer</summary>

```bash
k auth can-i create deployments --as=system:serviceaccount:dev:myuser -n dev
```

SA identity format: `system:serviceaccount:<namespace>:<name>`
</details>

---

### 15.7 Bind Role `pod-reader` to ServiceAccount `myuser` in namespace `dev` — [RoleBinding with SA](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-example)
<details><summary>answer</summary>

```bash
k create rolebinding pod-reader-sa --role=pod-reader --serviceaccount=dev:myuser -n dev
```
</details>

---

### 15.8 A pod using ServiceAccount `myuser` logs `Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:dev:myuser" cannot list resource "pods" in API group "" in the namespace "dev"`. Confirm the gap and fix it in place — [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
<details><summary>answer</summary>

```bash
k auth can-i list pods --as=system:serviceaccount:dev:myuser -n dev   # reproduces the denial instantly, no need to re-trigger the pod
k get rolebinding -n dev                                              # confirm myuser IS bound to some Role
k get role <role-name> -n dev -o yaml                                 # inspect rules[].verbs — e.g. only "get" is listed, not "list"
k edit role <role-name> -n dev                                        # add "list" (and "watch" if needed) to the verbs array
k auth can-i list pods --as=system:serviceaccount:dev:myuser -n dev   # confirm it now says "yes"
```
Most Forbidden errors on the exam are a missing verb/resource on an already-existing Role, not a
missing RoleBinding — check the Role's `rules` before assuming anything needs recreating.
</details>

---

## 16. Scheduling

> **Docs:** [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) · [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

### 16.1 Label node `node01` with `disktype=ssd`, then write a pod spec that uses `nodeSelector` to land only on it — [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)
<details><summary>answer</summary>

```bash
k label node node01 disktype=ssd
```

```yaml
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 16.2 Taint node `node01` with `tier=frontend:NoSchedule`, then write a pod spec that tolerates it — [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
<details><summary>answer</summary>

```bash
k taint node node01 tier=frontend:NoSchedule
```

```yaml
spec:
  tolerations:
  - key: tier
    operator: Equal
    value: frontend
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 16.3 Write a pod with required node affinity: only schedule on nodes labeled `region=eu-west` — [Node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
<details><summary>answer</summary>

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: region
            operator: In
            values:
            - eu-west
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 16.4 Write a pod with preferred node affinity: prefer nodes labeled `zone=az1` but don't require it — [Node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
<details><summary>answer</summary>

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - az1
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### 16.5 Remove the taint `tier=frontend:NoSchedule` from node `node01` — [Removing taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
<details><summary>answer</summary>

```bash
k taint node node01 tier=frontend:NoSchedule-
# trailing `-` removes the taint — same key/effect as when added
```
</details>

---

## 17. Kustomize

> **Docs:** [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
> Pattern: write `kustomization.yaml` in a directory → `kubectl apply -k <dir>` to deploy, `kubectl kustomize <dir>` to preview.

### 17.1 Apply all resources defined in a kustomization directory `./base` — [kubectl apply -k](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#how-to-apply-view-delete-objects-using-kustomize)
<details><summary>answer</summary>

```bash
k apply -k ./base
```
</details>

---

### 17.2 Write a `kustomization.yaml` referencing `deploy.yaml` and `svc.yaml`, adding namePrefix `dev-` and label `env: dev` to all resources — [Kustomization](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
<details><summary>answer</summary>

```yaml
# kustomization.yaml
resources:
- deploy.yaml
- svc.yaml

namePrefix: dev-

labels:
- pairs:
    env: dev
  includeSelectors: true   # also add it to selectors and pod templates
```

`labels` replaces the older `commonLabels` field, which still works but prints a deprecation
warning. Without `includeSelectors: true`, the label only goes on each object's own
`metadata.labels`. With it, kustomize also adds it to selectors and pod templates, which is what
`commonLabels` did. Adding it to selectors is fine for new objects but changes the selector of an
existing Deployment, and that field can't be changed after creation. To label pod templates but
leave selectors alone, use `includeTemplates: true` instead. Preview the result with
`k kustomize .`.
</details>

---

### 17.3 Write a kustomization overlay that patches the replica count of deployment `web` to 5 — [Patches](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#composing-and-customizing-resources)
<details><summary>answer</summary>

```yaml
# kustomization.yaml (overlay)
resources:
- ../base

patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web
    spec:
      replicas: 5
  target:
    kind: Deployment
    name: web
```

`patches` is the current field for both strategic-merge and JSON6902 patches. The older
`patchesStrategicMerge` and `patchesJson6902` are deprecated, and so is `bases:` (list a base
directory under `resources:`, as above). The patch can also live in its own file:
`patches: [{path: replicas-patch.yaml}]`. Match the target by the name used in the base, before
any `namePrefix` is added.
</details>

---

### 17.4 Write a kustomization that generates a ConfigMap `app-config` from a `.env` file — [ConfigMap generator](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#configmapgenerator)
<details><summary>answer</summary>

```yaml
# kustomization.yaml
configMapGenerator:
- name: app-config
  envs:
  - app.env
```

```bash
# app.env
ENV=prod
LOG_LEVEL=info
```

`envs:` turns each `KEY=VALUE` line into its own key (like `--from-env-file`); `files:` stores a
whole file under one key (like `--from-file`). The generated ConfigMap gets a content hash added to
its name (`app-config-dcd82k8b7k`), and kustomize updates references to it in the same
kustomization. Changing the data therefore changes the name and rolls the Deployments that use it.
Set `options: {disableNameSuffixHash: true}` on the generator if you need the plain name.
</details>

---

### 17.5 Preview what kustomize would generate without applying it — [kubectl kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
<details><summary>answer</summary>

```bash
k kustomize ./base
# or with the standalone tool:
kustomize build ./base
```
</details>

---

## 18. `kubectl get` — Filtering & Output Flags

> **Docs:** [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/) · [JSONPath support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
> `-l`/`--selector`, `-o`/`--output`, and `-L`/`--label-columns` are already drilled throughout
> (sections 1, 2, 6). This section fills in the ones that don't come up naturally elsewhere.

### 18.1 List only pods in phase `Running`, across all namespaces, using a field selector — [Field selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)
<details><summary>answer</summary>

```bash
k get pods -A --field-selector=status.phase=Running
```

`--field-selector` filters on a fixed, resource-specific set of built-in fields (`status.phase`,
`metadata.name`, `metadata.namespace`, ...) — not arbitrary jsonpath, and not custom labels. Compare
to `-l`, which filters on user-defined labels; the two are not interchangeable.
</details>

---

### 18.2 Show every pod's labels as a column, without knowing the label keys in advance — [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get pods --show-labels
```

`-L app,tier` only works if you already know which label keys to ask for — it prints just those as
columns. `--show-labels` dumps *all* labels on every object into one `LABELS` column, no key
knowledge required. Use `--show-labels` to discover what's there, `-L` once you know what you want.
</details>

---

### 18.3 List all pods across all namespaces, oldest first — [Sorting list objects](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get pods -A --sort-by=.metadata.creationTimestamp
```

`--sort-by` takes a jsonpath expression, evaluated against each returned object — not limited to
`kubectl top` or `kubectl get events` (where it's usually first seen). Works on any listable resource.
</details>

---

### 18.4 List all pods sorted by restart count, so the pods restarting most are easy to find — [Sorting list objects](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get pods --sort-by='.status.containerStatuses[0].restartCount'
```

`--sort-by` always sorts ascending and there's no `--reverse`, so the worst offenders end up at the
bottom: read the end of the list or pipe it through `tail`. `[0]` only looks at the first container.
For multi-container pods, check the RESTARTS column or the per-container counts in
`k describe`.
</details>

---

### 18.5 Get a script-friendly, one-name-per-line list of pods with label `app=web`, no header row — [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
<details><summary>answer</summary>

```bash
k get pods -l app=web --no-headers -o custom-columns=NAME:.metadata.name
```

`--no-headers` alone still leaves the default multi-column table; pairing it with
`-o custom-columns=NAME:.metadata.name` is what actually gets you one bare name per line — the
lower-risk alternative to jsonpath when you just need names to feed into another command.
`-o name` is shorter but prefixes each line with the type (`pod/web-1`), which is fine for piping
into `kubectl` but not when you need just the name.
</details>

---

## 19. `kubectl run` — Pod Creation Flags

> **Docs:** [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
> `--restart`, `--rm`, `--expose`, `-l`/`--labels`, `-o`/`--output` are already drilled in sections 1
> and 8. This section covers the remaining flags worth knowing cold.

### 19.1 Create pod `ann-test` from `nginx` with two annotations at creation time: `owner=alice` and `team=platform` — [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
<details><summary>answer</summary>

```bash
k run ann-test --image=nginx --annotations="owner=alice" --annotations="team=platform"
```

Gotcha: unlike `--labels`, which accepts a comma-separated list in one flag
(`--labels="a=1,b=2"` → two labels), `--annotations` does **not** split on commas — passing
`--annotations="owner=alice,team=platform"` produces exactly *one* annotation
(`owner: "alice,team=platform"`), not two. Repeat the flag per key/value instead.
</details>

---

### 19.2 Create pod `cmd-test` from `busybox`, replacing its ENTRYPOINT entirely (not just CMD) with `/bin/sh -c "echo hi"` — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run cmd-test --image=busybox --restart=Never --command -- /bin/sh -c "echo hi"
```

Without `--command`, whatever follows `--` populates the container's `args:` field — it overrides
the image's `CMD` but leaves `ENTRYPOINT` in place. With `--command`, the same tokens instead
populate `command:`, which overrides `ENTRYPOINT` itself. Matters when the base image's entrypoint
does something you specifically want to bypass (e.g. an entrypoint script that expects different args).
</details>

---

### 19.3 Create pod `env-test` from `busybox` with two env vars baked in at creation: `FOO=bar`, `BAZ=qux` — [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
<details><summary>answer</summary>

```bash
k run env-test --image=busybox --restart=Never --env="FOO=bar" --env="BAZ=qux"
```

Unlike `--annotations`, `--env` (like `--labels`) is designed to be repeated per variable — one
`NAME=value` pair per flag instance. Useful for quick one-off pods that need a value without writing
a ConfigMap/Secret first, e.g. passing a discovered IP: `--env="TARGET_IP=$TARGET_IP"`.
</details>

---

### 19.4 Create pod `priv-test` from `busybox` running as a privileged container — [Privileged pods](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
<details><summary>answer</summary>

```bash
k run priv-test --image=busybox --restart=Never --privileged
```

Sets `securityContext.privileged: true` on the container — gives it root-equivalent access to the
host (device access, kernel capabilities). Useful for a quick throwaway debug pod that needs host
access without hand-writing the full `securityContext` YAML; not something to leave running.
</details>

---

## Quick Reference — Fields You Must Not Have to Look Up

| What | Field path | Example |
|---|---|---|
| ConfigMap as env | `env[].valueFrom.configMapKeyRef` | `name: my-cm, key: my-key` |
| Secret as env | `env[].valueFrom.secretKeyRef` | `name: my-secret, key: my-key` |
| ConfigMap as volume | `volumes[].configMap.name` | — |
| Secret as volume | `volumes[].secret.secretName` | — |
| Init container | `spec.initContainers[]` | same fields as `containers[]` |
| Native sidecar | `spec.initContainers[].restartPolicy` | `Always` |
| Shared volume | `volumes[].emptyDir: {}` | — |
| PVC volume | `volumes[].persistentVolumeClaim.claimName` | — |
| Liveness exec | `livenessProbe.exec.command` | — |
| Liveness HTTP | `livenessProbe.httpGet.path/port` | — |
| Startup probe budget | `startupProbe.failureThreshold × periodSeconds` | `30 × 10` = 300s |
| Job completions | `spec.completions` | run N times sequentially |
| Job parallelism | `spec.parallelism` | run N pods at once |
| Job timeout | `spec.activeDeadlineSeconds` | — |
| CronJob miss window | `spec.startingDeadlineSeconds` | — |
| Node selector | `spec.nodeSelector` | `key: value` |
| Toleration | `spec.tolerations[]` | `key, operator, value, effect` |
| Service account | `spec.serviceAccountName` | — |
| Security capabilities | `securityContext.capabilities.add` | `["NET_ADMIN"]` |
| Run as user | `securityContext.runAsUser` | `1000` |
| Ingress backend | `spec.rules[].http.paths[].backend.service` | `name, port.number` |
| Ingress pathType | `spec.rules[].http.paths[].pathType` | `Prefix` / `Exact` / `ImplementationSpecific` (required; `kubectl create ingress` uses `Exact`) |
| Imperative ingress rule | `kubectl create ingress NAME --rule=...` | `"host/path=svc:port[,tls[=secret]]"` |
| Ingress rule path suffix | trailing `*` on the path | forces `pathType: Prefix` |
| NetPol deny-all ingress | `policyTypes: [Ingress]` + `ingress: []` | empty list = block all |
| NetPol deny-all egress | `policyTypes: [Egress]` + `egress: []` | empty list = block all |
| NetPol namespace | `from[].namespaceSelector.matchLabels` | namespace label selector |
| Resource requests | `resources.requests.memory/cpu` | `25Mi`, `500m` |
| Resource limits | `resources.limits.memory/cpu` | `100Mi`, `1` |
| Non-root enforce | `securityContext.runAsNonRoot` | `true` |
| Read-only FS | `securityContext.readOnlyRootFilesystem` | `true` |
| No privesc | `securityContext.allowPrivilegeEscalation` | `false` |
| fsGroup (pod) | `spec.securityContext.fsGroup` | `2000` |
| Disable SA token | `spec.automountServiceAccountToken` | `false` |
| Required affinity | `nodeAffinity.requiredDuringScheduling...` | `nodeSelectorTerms[].matchExpressions` |
| Preferred affinity | `nodeAffinity.preferredDuringScheduling...` | `weight + preference.matchExpressions` |
| DaemonSet update | `spec.updateStrategy.type` | `RollingUpdate` / `OnDelete` |
| RBAC role verb | `rules[].verbs` | `["get","list","watch"]` |
| RBAC resource | `rules[].resources` | `["pods"]` |
| SA identity (auth) | `system:serviceaccount:<ns>:<name>` | used with `--as=` |
| Switch namespace | `kubectl config set-context --current --namespace=X` | or pass `-n X` every time |
