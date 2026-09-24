# Scenarios

88 exam-style tasks you run against your own cluster. Each one gives you a starting state, a task,
and a worked solution with the reasoning behind it.

## How to use a scenario

1. Build the practice cluster from [../guide/practice-cluster.md](../guide/practice-cluster.md).
   Some scenarios need ingress-nginx or metrics-server, and they say so.
2. Run the **Setup** block. On the real exam this state would already exist ("there is a
   Deployment named X..."). Locally you have to create it first. Setup is scaffolding, not part of
   the task.
3. Read the **Task** and solve it without looking further.
4. Compare with the **Solution**. The notes under it explain the traps, which is where most of the
   learning is.
5. Run **Cleanup** so the scenario can be repeated.

Want to test yourself blind? The [mock exams](../mock-exams/) contain the same tasks without
solutions, under topic-neutral file names.

## Index

Grouped by [CKAD curriculum](https://github.com/cncf/curriculum) domain, with the domain's exam
weight.

### Application Design and Build (20%)

| # | Scenario | Difficulty |
|---|---|---|
| [004](004-job-completions-parallelism.md) | Job with completions, parallelism, and pod labels | Easy |
| [009](009-pod-to-deployment-conversion.md) | Convert a Pod into a Deployment with a container-level SecurityContext | Medium |
| [011](011-build-and-push-image.md) | Build with Docker and Podman, push to a registry, run detached, capture logs | Medium |
| [012](012-pv-pvc-deployment-mount.md) | PersistentVolume, matching PersistentVolumeClaim, and a Deployment that mounts it | Medium |
| [013](013-storageclass-pending-pvc.md) | StorageClass with an unfulfilled provisioner, capture the pending reason | Medium |
| [016](016-sidecar-tailing-log-file.md) | Add a native sidecar container that tails a shared log file | Hard |
| [017](017-init-container-shared-volume.md) | Init container that populates a shared volume before the main container starts | Medium |
| [026](026-cronjob-schedule.md) | Create a CronJob and inspect what it spawns | Easy |
| [027](027-daemonset-every-node.md) | DaemonSet running one Pod per node | Easy |
| [039](039-ambassador-container-pattern.md) | Ambassador container pattern: proxy localhost traffic to a real Service | Medium |
| [040](040-projected-volume.md) | Combine a ConfigMap and a Secret into one mounted directory with a projected volume | Medium |
| [041](041-statefulset-headless-service.md) | StatefulSet with stable per-Pod network identity via a headless Service | Medium |
| [042](042-taints-and-tolerations.md) | Keep ordinary Pods off a tainted node, admit only Pods that tolerate it | Medium |
| [046](046-job-active-deadline-seconds.md) | Kill a Job that runs too long, distinct from retrying a Job that fails | Medium |
| [059](059-offline-image-archive.md) | Hand over an image as a single archive for an offline site | Easy |
| [062](062-cronjob-activedeadlineseconds.md) | Put a hard runtime ceiling on every run a CronJob starts | Medium |
| [068](068-docker-build-run-push-save-combo.md) | Docker: build once, smoke-test on a published port, push two tags, bundle two names in one archive | Medium |
| [078](078-cronjob-history-limits-and-deadline-combo.md) | CronJob with a time zone, lopsided success/failure history limits and a per-run deadline | Medium |
| [087](087-statefulset-volumeclaimtemplates-ordered-scaledown.md) | StatefulSet with `volumeClaimTemplates`: a real PVC per Pod, and ordered shutdown | Medium |

### Application Deployment (20%)

| # | Scenario | Difficulty |
|---|---|---|
| [001](001-deployment-rollback.md) | Roll back a broken Deployment | Medium |
| [005](005-helm-release-operations.md) | Helm release operations (uninstall, upgrade, install with values, clean up stuck release) | Medium |
| [028](028-canary-deployment.md) | Canary deployment via replica-ratio traffic splitting | Hard |
| [029](029-kustomize-base-overlay.md) | Kustomize base + dev overlay with a replica-count patch | Medium |
| [036](036-poddisruptionbudget.md) | Protect a Deployment from voluntary disruption with a PodDisruptionBudget | Easy |
| [044](044-rolling-update-strategy-tuning.md) | Tune rolling-update strategy so a Deployment never dips below full capacity | Medium |
| [047](047-multi-document-yaml.md) | Author a ConfigMap, Deployment, and Service as one multi-document YAML file | Easy |
| [060](060-canary-clone-from-stable-deployment.md) | Clone an existing stable Deployment into a canary, don't author it from scratch | Medium |
| [066](066-deployment-rollback-undo-defaults-to-broken-revision.md) | Rollback trap: plain `rollout undo` can land you back on a *different* broken revision | Medium |
| [072](072-canary-with-total-pod-ceiling.md) | Canary release inside a Pod quota the stable version already exceeds | Medium |
| [077](077-rolling-update-surge-and-rollback.md) | Size a rollout's blast radius with maxSurge/maxUnavailable, push a bad tag, read the history, roll back | Medium |
| [086](086-blue-green-deployment-selector-cutover.md) | Blue/green deployment: instant full cutover via a Service selector patch | Medium |

### Application Observability and Maintenance (15%)

| # | Scenario | Difficulty |
|---|---|---|
| [007](007-readiness-probe.md) | Pod with an exec readiness probe | Easy |
| [031](031-api-deprecation-conversion.md) | Convert a manifest off a removed API version | Medium |
| [032](032-liveness-and-startup-probes.md) | Startup probe gating a slow app, liveness probe catching a later failure | Hard |
| [033](033-kubectl-debug-ephemeral-container.md) | Attach a debug shell to a running Pod with `kubectl debug` | Easy |
| [034](034-kubectl-top-resource-usage.md) | Flag the Pod holding the most memory with `kubectl top` | Easy |
| [035](035-horizontal-pod-autoscaler.md) | Autoscale a Deployment on CPU utilization | Medium |
| [037](037-multi-bug-triage.md) | Triage a Deployment broken by two unrelated, stacked bugs | Hard |
| [045](045-kubectl-cp-file-transfer.md) | Copy a file into and out of a running container | Easy |
| [054](054-crashloop-previous-logs.md) | CrashLoopBackOff: read the crash reason with `--previous`, not plain `logs` | Easy |
| [065](065-http-readiness-probe-existing-deployment.md) | Add an HTTP readiness probe to an already-running Deployment | Easy |
| [067](067-deprecated-cronjob-seccomp.md) | Revive an old CronJob manifest whose seccomp setting silently stopped working | Medium |
| [069](069-liveness-exec-probe-missing-command.md) | Liveness exec probe on a container whose command exits immediately | Easy |
| [076](076-deprecated-deployment-apiversion.md) | Convert a Deployment off a removed apiVersion, and the selector it never had | Medium |

### Application Environment, Configuration and Security (25%)

| # | Scenario | Difficulty |
|---|---|---|
| [006](006-serviceaccount-token.md) | Retrieve a ServiceAccount's token | Medium |
| [014](014-secret-env-and-volume-on-existing-pod.md) | Add a new Secret as env vars, and an existing Secret as a mounted volume, to a running Pod | Medium |
| [015](015-configmap-for-pending-deployment.md) | Create the missing ConfigMap a Deployment is already waiting on | Easy |
| [021](021-deployment-serviceaccount-resources.md) | Deployment running under a specific ServiceAccount with memory requests/limits | Easy |
| [023](023-rbac-role-rolebinding.md) | Restrict a ServiceAccount to read-only Pod access with a Role + RoleBinding | Medium |
| [024](024-capabilities-add-drop.md) | Drop all Linux capabilities except the ones a container actually needs | Hard |
| [025](025-crd-custom-resource.md) | Discover a CRD and create a Custom Resource instance | Medium |
| [038](038-secure-microservice-rbac-networkpolicy.md) | Harden a microservice: scoped RBAC plus network egress restriction together | Hard |
| [043](043-limitrange-defaults.md) | LimitRange auto-injecting resource requests/limits on bare Pods | Easy |
| [048](048-rbac-forbidden-diagnosis.md) | Diagnose and fix a ServiceAccount hitting `Forbidden`, without recreating anything | Medium |
| [051](051-securitycontext-merge-existing-deployment.md) | Add `runAsUser` to an existing Deployment without losing its other securityContext fields | Medium |
| [052](052-resourcequota-guaranteed-qos.md) | A quota that insists on limits: get a Deployment's Pods created, with Guaranteed QoS | Medium |
| [053](053-convert-hardcoded-env-to-secret.md) | Convert a hardcoded env var to a Secret, without touching the other env vars | Medium |
| [058](058-resourcequota-edit-existing-limits-scale.md) | Over-sized limits block a scale-out under a quota: shrink, fix, then grow | Hard |
| [063](063-securitycontext-merge-nested-capabilities.md) | Set a container's UID and GID without losing its existing securityContext, in a two-container Pod | Easy |
| [064](064-rbac-create-from-scratch-forbidden.md) | RBAC from scratch: create SA, Role, RoleBinding, and wire them to a Deployment | Hard |
| [071](071-immutable-pod-securitycontext-delete-recreate.md) | Harden a bare Pod whose securityContext can't be changed in place | Medium |
| [075](075-resourcequota-headroom-top-cpu.md) | Size a Pod to the quota headroom that's left, then find the top CPU consumer | Medium |
| [079](079-runasnonroot-enforcement.md) | `runAsNonRoot: true` rejects an image that is already non-root | Medium |
| [080](080-pod-security-standards-restricted.md) | Pod Security Admission: preview, enforce and fix a Deployment under `restricted` | Medium |
| [082](082-limitrange-min-max-rejection.md) | A Pod rejected by a LimitRange's max and limit/request ratio, fixed to the largest limits allowed | Easy |
| [084](084-rbac-swap-to-correct-serviceaccount.md) | RBAC fix without touching RBAC: pick the right ServiceAccount out of several | Medium |
| [085](085-qos-class-and-oomkilled-diagnosis.md) | The three QoS classes, and what an OOM kill actually looks like on this cluster | Medium |
| [088](088-imagepullbackoff-missing-pull-secret.md) | ImagePullBackOff from a private registry, fixed with `imagePullSecrets` | Medium |

### Services and Networking (20%)

| # | Scenario | Difficulty |
|---|---|---|
| [010](010-clusterip-service-from-scratch.md) | ClusterIP Service exposing a new Pod, tested with a temporary curl Pod | Easy |
| [018](018-service-selector-mismatch.md) | Diagnose and fix a Service with a mismatched selector | Medium |
| [019](019-clusterip-to-nodeport.md) | Convert a ClusterIP Service to NodePort, test via node IP | Easy |
| [020](020-networkpolicy-egress-restriction.md) | Restrict egress from one Deployment to only another, plus DNS | Hard |
| [030](030-ingress-host-routing.md) | Ingress routing a host to a Service | Medium |
| [049](049-ingress-wrong-backend-fix.md) | Fix an Ingress pointing at the wrong Service, without recreating it | Medium |
| [050](050-networkpolicy-label-fix.md) | Restore connectivity blocked by an existing NetworkPolicy, by relabeling the client Pod | Easy |
| [055](055-ingress-path-based-multi-service.md) | One Ingress, one host, two Services split by path | Medium |
| [056](056-ingress-multi-host-different-services.md) | One Ingress, multiple hosts, each to a different Service | Medium |
| [057](057-networkpolicy-multi-policy-three-pod-chain.md) | Egress and ingress policies both gate a call: fix two clients by relabeling only | Hard |
| [061](061-service-selector-label-key-mismatch.md) | Service selector using the wrong label *key*, not just the wrong value | Medium |
| [070](070-networkpolicy-one-pod-two-independent-policies.md) | Let a Deployment through two existing NetworkPolicies by labels alone | Medium |
| [073](073-deployment-label-scope-trap-nodeport.md) | A Service selecting on a label that `kubectl label deployment` never gave the Pods | Medium |
| [074](074-networkpolicy-ipblock-except-and-peer-or-trap.md) | Fix two drafted NetworkPolicies: a `from` list that ORs, and an egress `ipBlock` without DNS | Hard |
| [081](081-kubectl-port-forward.md) | Reach a Pod (or Service) directly with `kubectl port-forward`, no Service required | Easy |
| [083](083-networkpolicy-cross-namespace-selector.md) | Cross-namespace NetworkPolicy via `namespaceSelector`, and the port it actually checks | Hard |

### Core kubectl

Not a curriculum domain, but exam mechanics every question relies on.

| # | Scenario | Difficulty |
|---|---|---|
| [002](002-list-namespaces-to-file.md) | List namespaces to a file | Easy |
| [003](003-pod-status-script.md) | Create a Pod and a reusable status-check script | Easy |
| [008](008-move-pod-between-namespaces.md) | Move a Pod to a different namespace | Medium |
| [022](022-label-annotate-matching-pods.md) | Label and annotate Pods based on an existing label match | Easy |

## Sources

The scenarios are written in my own words, with their own names and values. Many are modelled on
task *types* reported by people who took the exam, and a few on practice-question collections:

- Exam write-ups: [codebob75](https://medium.com/@codebob75/passing-ckad-cheatsheet-notes-and-tips-1aa285e6a473),
  [codegenitor](https://codegenitor.medium.com/i-passed-ckad-heres-what-actually-came-up-and-what-helped-me-after-failing-once-fb5914b15f22),
  [Lajko on ITNEXT](https://itnext.io/how-the-ckad-certification-has-changed-from-2021-to-2024-06aec019a35a),
  [ExamCert](https://www.examcert.app/blog/ckad-practice-questions-tips-2026/). Scenarios 048–088
  in particular cover the task types these describe.
- Practice exams: the killer.sh CKAD simulator and MyExamCloud's free CKAD practice tests, which
  inspired the task types of several scenarios in 001–047.

If you want the original questions, go to those sources; none are reproduced here.

## Format

Every file follows the same shape:

```
# NNN — Title
**Domain:** <curriculum domain> · **Difficulty:** Easy | Medium | Hard

## Task            the question, as the exam would phrase it
## Documentation   which kubernetes.io (or helm.sh) page to look it up on
## Setup           commands that create the starting state
## Solution        worked answer, with explanations of the gotchas
## Cleanup         delete what Setup created
```

Conventions:

- **No hardcoded generated names.** Pod names with hashes are looked up in the step that needs
  them (`POD=$(kubectl get pods -l app=x -o jsonpath='{.items[0].metadata.name}')`), so every
  block can be pasted as is. You'll need the same habit in the exam.
- **One namespace per scenario**, so scenarios don't interfere with each other.
- **Writable paths.** Exam questions write answer files to paths like `/opt/...`; here they go to
  `~/ckad/NNN/` so you don't need root.
- **kind differences are called out.** Where the local cluster behaves differently from an exam
  cluster (NetworkPolicy enforcement, the default StorageClass, missing metrics-server), the
  scenario says so instead of failing silently.
- **Verified.** Every scenario was run end to end on a live kind cluster when it was written. Where
  a footer is present, it records the most recent full run.
