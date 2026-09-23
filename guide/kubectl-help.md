# kubectl help and kubectl explain

If there's one thing I'd tell you to learn properly, it's this. In the exam you have the Kubernetes
docs in a browser, but the fastest reference is already in the terminal. `kubectl <command> -h`
tells you how to *create* something, and `kubectl explain` tells you where a field *goes*. Between
them they answer most of the "what was that flag / where does that field live" moments, without
leaving the keyboard or scrolling a docs page.

In the exam I kept two terminals open: one to run commands, the other to check files and look
things up with `-h` and `explain` (see [my exam setup](exam-tips.md#my-exam-setup)).

- [`kubectl -h`: how to create things](#kubectl--h-how-to-create-things)
- [`kubectl explain`: where fields go](#kubectl-explain-where-fields-go)
- [Putting them together](#putting-them-together)
- [Drill it](#drill-it)

---

## `kubectl -h`: how to create things

### Start from the list
`kubectl create -h` lists everything `create` can make imperatively:

```
Available Commands:
  clusterrole, clusterrolebinding, configmap, cronjob, deployment, ingress, job, namespace,
  poddisruptionbudget, priorityclass, quota, role, rolebinding, secret, service,
  serviceaccount, token
```

Two things that aren't in that list, because they have their own commands:
- **Pods** are created with `kubectl run` (there's no `kubectl create pod`).
- **Services from an existing object** come from `kubectl expose`.

Some entries have their own sub-commands, and `-h` works at every level:

```bash
kubectl create secret -h            # docker-registry, generic, tls
kubectl create secret generic -h    # --from-literal, --from-file, --from-env-file, examples
kubectl create service -h           # clusterip, nodeport, loadbalancer, externalname
kubectl set -h                      # env, image, resources, selector, serviceaccount, subject
kubectl rollout -h                  # history, pause, restart, resume, status, undo
```

### Read the Examples section first
Every `-h` page has an **Examples** block near the top with complete, working commands. It's usually
faster to copy one and change the names than to read the flag list:

```
$ kubectl create deployment -h
Examples:
  # Create a deployment named my-dep that runs the nginx image with 3 replicas
  kubectl create deployment my-dep --image=nginx --replicas=3

  # Create a deployment named my-dep that runs the busybox image and expose port 5701
  kubectl create deployment my-dep --image=busybox --port=5701
  ...
```

Where this saves the most time, because the syntax is hard to remember:

| Command | What the examples show you |
|---|---|
| `kubectl create ingress -h` | the `--rule="host/path=service:port"` syntax, `path*` for Prefix, `tls=secret` |
| `kubectl create job -h` | `--from=cronjob/<name>` to trigger a CronJob by hand |
| `kubectl create cronjob -h` | `--schedule` quoting and a command after `--` |
| `kubectl create role -h` / `rolebinding -h` | `--verb`, `--resource`, `--serviceaccount=ns:name` |
| `kubectl create secret tls -h` | `--cert` / `--key` |
| `kubectl run -h` | `--command` vs args after `--`, `--env`, `--labels`, `--rm -it` |
| `kubectl expose -h` | `--port` vs `--target-port`, `--type=NodePort` |
| `kubectl rollout undo -h` | `--to-revision` |
| `kubectl autoscale -h` | `--min`, `--max`, CPU target |
| `kubectl taint -h`, `kubectl label -h` | the trailing `-` that *removes* a taint or label |

### Then search the flags
The options list is long. `grep` for the flag you half-remember; anchoring on leading spaces
matches the flag's own entry rather than every example that uses it:

```bash
kubectl run -h | grep -A1 -E '^\s+--restart'
kubectl create deployment -h | grep -A1 -E '^\s+--port'
kubectl logs -h | grep -A1 -E '^\s+--(previous|all-containers|since)'
```

### Generate the YAML instead of writing it
The imperative command plus `--dry-run=client -o yaml` gives you a correct skeleton for
everything you then edit by hand (probes, volumes, securityContext):

```bash
kubectl create deployment web --image=nginx --replicas=2 --dry-run=client -o yaml > web.yaml
kubectl create cronjob backup --image=busybox --schedule="0 2 * * *" --dry-run=client -o yaml -- sh -c 'echo backup'
```

---

## `kubectl explain`: where fields go

`-h` is about commands. `explain` is about the **API schema**: which fields a resource has, their
types, and where they're nested. It reads the schema from the cluster you're connected to, so it
always matches the exam's Kubernetes version.

### Use the full dotted path
You don't need to go one level at a time. Jump straight to the field:

```
$ kubectl explain pod.spec.containers.securityContext
KIND:       Pod
VERSION:    v1

FIELD: securityContext <SecurityContext>

DESCRIPTION:
    SecurityContext defines the security options the container should be run
    with. If set, the fields of SecurityContext override the equivalent fields
    of PodSecurityContext. ...

FIELDS:
  allowPrivilegeEscalation  <boolean>
  appArmorProfile           <AppArmorProfile>
  capabilities              <Capabilities>
  ...
```

Short names work too: `kubectl explain deploy.spec.strategy`, `kubectl explain netpol.spec`,
`kubectl explain cj.spec.jobTemplate`.

### `--recursive` for the whole shape at once
`--recursive` prints only field names and types, indented as they nest. It's the fastest way to see
the full shape of something before writing YAML:

```
$ kubectl explain deploy.spec.strategy --recursive
FIELDS:
  rollingUpdate  <RollingUpdateDeployment>
    maxSurge        <IntOrString>
    maxUnavailable  <IntOrString>
  type  <string>
  enum: Recreate, RollingUpdate
```

Combine it with `grep` when you remember a field's name but not where it lives:

```bash
kubectl explain pod.spec --recursive | grep -n -i toleration
kubectl explain pod.spec --recursive | grep -n -i runAsNonRoot   # appears at several levels: pod and container
```

Then confirm the exact path with a plain `explain`
(`kubectl explain pod.spec.securityContext.runAsNonRoot` vs
`kubectl explain pod.spec.containers.securityContext.runAsNonRoot`).

### Questions `explain` answers well
- **Pod level or container level?** `securityContext` exists at both, but `capabilities` only at
  container level and `serviceAccountName` only at pod level.
- **What are the allowed values?** Look for `ENUM:` or "Possible enum values" (e.g. `pathType`,
  `restartPolicy`, `imagePullPolicy`, `strategy.type`).
- **Is it a list or a map?** `<[]Toleration>` is a list (`- key: ...`), `<map[string]string>` is a
  map (`key: value`), `<Object>` is a nested block.
- **What does this field default to?** The description often says ("Defaults to 3").
- **Which API version?** The header shows `GROUP` and `VERSION`, which is handy for
  deprecation questions. `kubectl api-resources` lists everything the cluster knows, with short
  names and groups.

---

## Putting them together

A typical flow for "create a Deployment with a readiness probe and a non-root security context":

```bash
# 1. skeleton from the imperative command (other terminal: kubectl create deployment -h if unsure)
kubectl create deployment api --image=nginx --port=80 --dry-run=client -o yaml > api.yaml

# 2. where do probes and securityContext go?
kubectl explain deploy.spec.template.spec.containers.readinessProbe.httpGet
kubectl explain deploy.spec.template.spec.securityContext --recursive

# 3. edit api.yaml, then check it without creating anything
kubectl apply -f api.yaml --dry-run=server
kubectl apply -f api.yaml
```

`--dry-run=server` validates the YAML against the real API. A typo'd field name or a wrong
nesting level fails there, before anything is created.

---

## Drill it

This is a habit, and it only sticks if you use it while practising:
- Work through the [drills](../drills/drill.md) and [scenarios](../scenarios/) **without the
  browser docs**, only `-h` and `explain`. Open the docs only when those really don't cover it
  (e.g. a full NetworkPolicy example).
- Drill section 1 (1.10–1.12) has questions specifically on `explain` and `-h`.
- When you look something up in the docs, ask afterwards whether `-h` or `explain` would have
  been faster. Usually it would.
