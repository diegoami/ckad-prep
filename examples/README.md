# Example manifests

Small, working manifests to `apply`, break and inspect while you drill. They're plain examples
rather than exam questions; for those, see [../scenarios/](../scenarios/).

Try them on the practice cluster from [../guide/practice-cluster.md](../guide/practice-cluster.md),
in a scratch namespace:

```bash
kubectl create ns play && kubectl config set-context --current --namespace=play
```

## Pods

| File | Shows |
|---|---|
| [pods/multi-container-pod.yaml](pods/multi-container-pod.yaml) | Two containers in one Pod (`kubectl exec -c`, `kubectl logs -c`) |
| [pods/init-container-pod.yaml](pods/init-container-pod.yaml) | An init container writing to a shared `emptyDir` that nginx then serves |
| [pods/configmap-volume-pod.yaml](pods/configmap-volume-pod.yaml) | A ConfigMap mounted as files (create `app-config` first) |
| [pods/secret-env-pod.yaml](pods/secret-env-pod.yaml) | One Secret key as an env var (create `db-creds` first) |
| [pods/downward-api-volume.yaml](pods/downward-api-volume.yaml) | The Pod's own labels and annotations exposed as files with a `downwardAPI` volume; relabel it and watch the file change |

```bash
kubectl create configmap app-config --from-literal=color=blue
kubectl create secret generic db-creds --from-literal=password=s3cret
```

## Jobs and CronJobs

| File | Shows |
|---|---|
| [jobs/job-completions.yaml](jobs/job-completions.yaml) | `completions: 5`, run one after another |
| [jobs/job-parallel.yaml](jobs/job-parallel.yaml) | `completions: 5` with `parallelism: 5` |
| [jobs/job-deadline.yaml](jobs/job-deadline.yaml) | `activeDeadlineSeconds` |
| [jobs/indexed-job.yaml](jobs/indexed-job.yaml) | `completionMode: Indexed`: 4 shards, 2 at a time, each Pod reading its shard from `$JOB_COMPLETION_INDEX` |
| [jobs/cronjob.yaml](jobs/cronjob.yaml) | A CronJob running every minute |
| [jobs/cronjob-starting-deadline.yaml](jobs/cronjob-starting-deadline.yaml) | `startingDeadlineSeconds` |

## Deployments

| File | Shows |
|---|---|
| [deployments/canary.yaml](deployments/canary.yaml) | Canary: `v1` (3 replicas) and `v2` (1 replica) behind one Service through a shared label, so about 25% of traffic reaches v2 |

## StatefulSet

[statefulset/statefulset-volumeclaimtemplates.yaml](statefulset/statefulset-volumeclaimtemplates.yaml)
is a 3-replica Postgres StatefulSet whose `volumeClaimTemplates` give each Pod its own PVC
(`data-bob-db-0`, `-1`, `-2`). Scale it down and back up to see the PVCs survive.

## NetworkPolicy

Both policies use namespace `netpol-demo` (`kubectl create ns netpol-demo`).

| File | Shows |
|---|---|
| [networkpolicy/allow-ingress-from-label.yaml](networkpolicy/allow-ingress-from-label.yaml) | `app=frontend` Pods accept ingress only from Pods labelled `frontend=true` |
| [networkpolicy/allow-ingress-and-egress.yaml](networkpolicy/allow-ingress-and-egress.yaml) | `app=web` Pods talk only to Pods labelled `webaccess=true`, both directions. No DNS rule, so name lookups break; try it |

## Ingress

[ingress/basic/](ingress/basic/) contains a Deployment, a Service on port 99 → 80, and an Ingress
without a host that routes `/` to it. It needs the ingress-nginx controller from the practice-cluster
guide.

```bash
kubectl apply -f ingress/basic/
curl http://localhost/
```

## Kustomize

Each directory is self-contained and built around a small fictional bookshop app. Preview with
`kubectl kustomize <dir>`, apply with `kubectl apply -k <dir>`. Together they cover the features
described on the [kubernetes.io Kustomize page](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/),
the only Kustomize reference you'll have during the exam.

| Directory | Shows |
|---|---|
| [01-configmap-generator](kustomize/01-configmap-generator/) | `configMapGenerator` from a file, an env file and literals; compare the three results |
| [02-configmap-in-deployment](kustomize/02-configmap-in-deployment/) | A generated ConfigMap mounted in a Deployment; the hash suffix is rewritten in the reference for you |
| [03-secret-generator](kustomize/03-secret-generator/) | `secretGenerator` mixing literals and a file, with an explicit key name for the file |
| [04-secret-in-deployment](kustomize/04-secret-in-deployment/) | A generated Secret consumed with `envFrom` |
| [05-generator-options](kustomize/05-generator-options/) | `generatorOptions`: no hash suffix, extra labels and annotations |
| [06-namespace-prefix-labels](kustomize/06-namespace-prefix-labels/) | `namespace`, `namePrefix`, `nameSuffix`, `labels`, `commonAnnotations` |
| [07-strategic-merge-patches](kustomize/07-strategic-merge-patches/) | `patches:` with partial manifests: scale up, add requests/limits |
| [08-images](kustomize/08-images/) | `images:` to change image name and tag without editing the manifest |
| [09-json-patch](kustomize/09-json-patch/) | `patches:` with a JSON 6902 patch (replace replicas, add an env var) and an explicit `target` |
| [10-replacements](kustomize/10-replacements/) | `replacements:` copying a Service's final (prefixed) name into an app's env var |
| [11-base-and-overlays](kustomize/11-base-and-overlays/) | A `base/` with `dev/` and `prod/` overlays, prod scaled with an inline patch; apply an overlay, not the base |
| [12-labels-include-selectors](kustomize/12-labels-include-selectors/) | `labels` with and without `includeSelectors` (what the old `commonLabels` did) |
