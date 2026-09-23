# CKAD exam tips

Things I learned preparing for (and passing) the CKAD that the curriculum doesn't tell you: what
the curriculum really expects, and the gotchas that cost the most time under pressure.

For recurring question *shapes* (blue/green, ConfigMap injection, PV/PVC, and so on), see
[task-patterns.md](task-patterns.md). For drilling individual commands, see
[../drills/drill.md](../drills/drill.md).

**Contents**

- [My exam setup](#my-exam-setup)
- [What the curriculum actually tests](#what-the-curriculum-actually-tests)
- [Users vs ServiceAccounts](#users-vs-serviceaccounts)
- [Output: jsonpath, not Go templates](#output-jsonpath-not-go-templates)
- [Init containers: write paths must be absolute](#init-containers-write-paths-must-be-absolute)
- [Service discovery and endpoints](#service-discovery-and-endpoints)
- [NetworkPolicy egress gotchas](#networkpolicy-egress-gotchas)
- [Imperative flag gotchas](#imperative-flag-gotchas)
- [Exploring the cluster without leaving the terminal](#exploring-the-cluster-without-leaving-the-terminal)
- [kubectl debug](#kubectl-debug)
- [Handy one-liners](#handy-one-liners)

---

## My exam setup

**One VM per question.** Every question has you `ssh` into its own VM, so there are as many
machines as there are questions. Anything you configure on one of them is gone at the next one.
Setting up vim or installing tmux on every VM isn't worth the time; any configuration you do is a
waste.

**Screen layout.** The question takes up the left part of the screen, and you can hide it when you
need the space. On the rest I had:

```
+------------------+------------------------------------------------+
|                  |  browser (kubernetes.io docs)                  |
|  question        |                                                |
|  (can be hidden) +-----------------------+------------------------+
|                  |  terminal 1           |  terminal 2            |
|                  |  running commands     |  checking files, and   |
|                  |                       |  docs with kubectl -h  |
+------------------+-----------------------+------------------------+
```

One terminal to execute, the other to check files and look things up with `kubectl ... -h` and
`kubectl explain`. Learning to use those two well is the most useful preparation I can recommend;
see [kubectl-help.md](kubectl-help.md).

**Two attempts, and a practice environment.** The exam comes with two attempts, and the first one
is also your chance to learn the environment for real. If you buy the exam through the Linux
Foundation, you also get access to a practice environment for two days beforehand. It gets you used
to Xfce, but the real exam environment is worse:
- **The browser is slower in the real exam** than in the practice environment.
- **The practice environment doesn't prepare you for the one-VM-per-question setup.** In the exam,
  every single question is answered on a different cluster, on a different VM you `ssh` into. So
  any configuration you do is a waste.

**Know your tools until they're second nature.** Under time pressure there's no room to wonder how
the terminal copies and pastes, how to open a second one, or what it can't do. To get used to it
before the exam, I installed Xfce on Ubuntu and practised in its terminal until using it was second
nature.

**vim.** You can't know vim too well, unless you go with nano instead. Learn what most vim
tutorials teach you, but above all learn how to **indent and unindent**, because in YAML the
indentation *is* the structure:

| Keys | What it does |
|---|---|
| `>>` / `<<` | indent / unindent the current line |
| `3>>` / `3<<` | indent / unindent 3 lines, starting at the cursor |
| `V`, move with `j`/`k`, then `>` or `<` | indent / unindent a visual block of lines |
| `.` | repeat the last indent (press it again to move the block one more step) |
| `gv` | reselect the last visual block |
| `u` / `Ctrl-r` | undo / redo |

Plain vim indents by one tab of 8 columns, which breaks YAML. If `>>` jumps that far,
`:set sw=2 et` fixes it for the file you have open.

The basics most tutorials cover are worth having in your fingers too: `i`/`Esc`, `:wq` and `:q!`,
`dd`/`yy`/`p`, `/text` then `n` to search, `x`, and `:set paste` before pasting YAML from the
browser, so the indentation doesn't cascade.

---

## What the curriculum actually tests

The [curriculum](https://github.com/cncf/curriculum) is short and vague. Here's how I read the
fuzzier lines.

### Helm: "deploy existing packages"
- "Existing packages" means the chart is already reachable: a repo is likely configured already, or
  you get a local `.tgz`.
- Focus on `helm install`, `helm upgrade`, `helm uninstall`, `helm list` (`-A`, `--pending`),
  `helm rollback`, `helm history`, `helm get values`, `helm show values`, `helm pull --untar`.
- `helm repo add` / `helm repo update` / `helm search repo` are less likely to be the point of a
  question, but know them.
- Writing chart templates is not in scope.

### CRDs and Operators: "discover and use"
- The word is *discover and use*, not *define* or *implement*.
- Expect `kubectl get crd`, `kubectl describe crd <name>`, `kubectl api-resources`, creating a custom
  resource from a CRD that already exists, and listing CRs by short name.
- Writing a full CRD (`openAPIV3Schema`, `versions[]`, ...) is cluster-admin work and unlikely.

### Kustomize: shallow by necessity
- There's essentially one page about it on `kubernetes.io/docs`, so the exam can't test deep
  knowledge.
- Stick to `kustomization.yaml` with `resources`, `namePrefix`/`nameSuffix`, `labels`/`commonLabels`,
  `images`, `configMapGenerator`/`secretGenerator`, simple `patches`, base + overlay directories,
  `kubectl apply -k` and `kubectl kustomize`.
- `kubectl.docs.kubernetes.io` (the full Kustomize reference) isn't on the allowed list.

### Podman: build, run, push
- The curriculum says *"Define, build and modify container images"*.
- Most valuable: writing a Dockerfile, `podman build -t`, `podman run -d -p`, `podman tag`,
  `podman push`, `podman save -o` (image to tar).
- `podman <cmd> --help` works in the terminal, and it's the same CLI as `docker`. Podman docs are
  only available when a question links them.

### `podman save` vs `podman export`
Both write a `.tar`, but they aren't interchangeable. `podman export` (drill 12.4) dumps a
*container's* filesystem as a flat tar with no image manifest or layers, and it can't be
`podman load`ed back as an image. `podman save` (drill 12.7) writes the whole *image*. For "save the
image as a tar in OCI format", use `podman save --format oci-archive -o file.tar <image>`: the
default format is `docker-archive`, so `--format` must be explicit.

### Dockerfile instructions: memorise them, there's nothing to look up
Even when `docs.podman.io` is available, it has **no page documenting Dockerfile/Containerfile
instruction syntax**. The `Containerfile(5)` man page it links to lives on GitHub, which is blocked.
`podman build --help` only covers build *flags*. So learn these:

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | base image | `FROM node:20-alpine` |
| `RUN` | run a command at build time (creates a layer) | `RUN npm install --production` |
| `COPY` | copy files from the build context into the image | `COPY package.json ./` |
| `ADD` | like `COPY`, plus tar auto-extraction and remote URLs | `ADD app.tar.gz /opt/app/` |
| `WORKDIR` | working directory for later instructions and the container | `WORKDIR /app` |
| `ENV` | environment variable baked into the image | `ENV NODE_ENV=production` |
| `ARG` | build-time-only variable, absent from the final image | `ARG VERSION=1.0` |
| `EXPOSE` | documents the listening port (metadata only, publishes nothing) | `EXPOSE 8080` |
| `USER` | user (name or UID) for later instructions and the container | `USER 1000` |
| `VOLUME` | declares a mount point | `VOLUME /var/lib/data` |
| `LABEL` | key=value metadata on the image | `LABEL maintainer="team@example.com"` |
| `CMD` | default command/args, overridable at `run` | `CMD ["npm", "start"]` |
| `ENTRYPOINT` | fixed executable; `CMD` becomes its default args | `ENTRYPOINT ["node"]` |

```dockerfile
FROM node:20-alpine
LABEL maintainer="team@example.com"
ARG VERSION=1.0
ENV NODE_ENV=production
WORKDIR /app
COPY package.json ./
RUN npm install --production
COPY . .
EXPOSE 8080
USER 1000
ENTRYPOINT ["node"]
CMD ["server.js"]
```
`podman run myimage` runs `node server.js`; `podman run myimage worker.js` replaces only the `CMD`
part and runs `node worker.js`.

### CronJob schedule syntax: memorise it too
No allowed site documents crontab field order well. The order is
minute, hour, day-of-month, month, day-of-week. `*/15 * * * *` means every 15 minutes, and
`0 2 * * 1-5` means 02:00 on weekdays. See [drill 4.14](../drills/drill.md) for the diagram.

### yq: available, but don't depend on it
`yq` may be installed on the exam machines, but its docs aren't allowed, so if you blank on the
syntax you have no reference. Prefer `kubectl set image|env|resources|serviceaccount`,
`kubectl patch`, `kubectl label`/`annotate`, `kubectl scale`, all documented on `kubernetes.io`,
and use `kubectl edit` (or vim on the YAML file) for everything else. If you do use yq, [yq-vs-jsonpath.md](yq-vs-jsonpath.md)
maps the common jsonpath queries to yq.

### Gateway API: not in the curriculum
The curriculum says *"Use Ingress rules to expose applications"*. Know Ingress well. Gateway API
needs a controller and is operations-focused.

---

## Users vs ServiceAccounts

### Kubernetes has no user objects
- There's no `kubectl create user`. Users are external: X.509 certificates, OIDC tokens, kubeconfig
  entries.
- A RoleBinding stores a username as a plain **string**, and Kubernetes doesn't check it exists.
- Test permissions for any name with impersonation:
  `k auth can-i list pods --as=alice -n dev`

### ServiceAccounts are real, namespaced objects
- `k create sa myapp` creates one. Pods run as a ServiceAccount (`default` unless you set
  `serviceAccountName` on the pod spec, or on the pod *template* spec of a Deployment).
- Exam RBAC questions almost always use ServiceAccounts rather than users.
- The ServiceAccount's identity for `--as` is `system:serviceaccount:<namespace>:<name>`:

```bash
k auth can-i create deployments --as=system:serviceaccount:dev:myapp -n dev
k auth can-i --list --as=system:serviceaccount:dev:myapp -n dev
```

---

## Output: jsonpath, not Go templates

Use **jsonpath**. It's documented at `kubernetes.io/docs/reference/kubectl/jsonpath/`; Go
template syntax isn't documented anywhere you can reach during the exam.

```bash
k get pod nginx -o jsonpath='{.status.podIP}{"\n"}'
k get secret my-secret -o jsonpath='{.data.mykey}' | base64 -d
k get nodes -o jsonpath='{.items[*].metadata.name}'
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

- Append `{"\n"}` so the output doesn't run into your next prompt.
- Type the whole `-o jsonpath='...'` on one line. An accidental Enter before the closing quote
  embeds a newline in the expression.
- If jsonpath is the sticking point, `-o name` or `--no-headers -o custom-columns=...` give you
  names and fields with much less quoting to get wrong:

```bash
k get pods -l app=web -o name
k get pods -l app=web --no-headers -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

---

## Init containers: write paths must be absolute

An init container's shell command runs in its own working directory (usually `/`), which has
nothing to do with its `volumeMounts`. `command: ['sh', '-c', 'echo hi > index.html']` writes to the
container's cwd, **not** into the mounted volume. The main container then serves missing content,
and the failure looks like a mount or timing problem rather than a scripting one. Always write to
the full mounted path:

```yaml
command: ['sh', '-c', 'echo "hello" > /usr/share/nginx/html/index.html']
```

Init containers run only once, when the pod is created. After you fix the command in a Deployment,
existing pods won't rerun it. `kubectl rollout restart deploy <name>` forces new pods. See
[scenario 017](../scenarios/017-init-container-shared-volume.md).

---

## Service discovery and endpoints

### DNS resolves Services, not Deployments or Pods
`curl http://<deployment-name>` fails (hangs or returns NXDOMAIN) even when matching pods are
healthy. Cluster DNS only creates records for Services (plus pod records under headless Services),
never for Deployment, ReplicaSet or Pod names. Check a Service exists before testing by name:

```bash
k get svc
k expose deploy <name> --port=80 --target-port=80   # if it doesn't
k run tmp --rm -it --restart=Never --image=busybox -- wget -qO- http://<svc>.<ns>.svc.cluster.local
```

### `kubectl get endpoints` separates "Service misconfigured" from everything else
When a Service exists but traffic doesn't get through, run `k get endpoints <svc>` (or
`k get endpointslices -l kubernetes.io/service-name=<svc>`) first. It shows exactly which pod IPs
the selector matches.
- **Empty** means the selector matches no ready pod: a wrong label value, a typo, or selecting on
  the Deployment's own labels instead of the pod template's labels.
- **Not empty** means the Service is wired correctly and the problem is elsewhere: a wrong
  `targetPort`, pod readiness, a NetworkPolicy, or the app itself.

---

## NetworkPolicy egress gotchas

### List nesting under `egress`/`ingress` changes the meaning
Each top-level `-` under `egress:` is a **separate rule**, and rules are OR'ed together. Whether
`ports:` sits in the same list item as `to:` or in its own item changes what's allowed:

```yaml
egress:
- to:                     # rule 1: all traffic (any port) to `api` pods
  - podSelector:
      matchLabels: {id: api}
- ports:                  # rule 2: DNS (53/UDP+TCP) to any destination
  - port: 53
    protocol: UDP
  - port: 53
    protocol: TCP
```

Nest `ports:` inside the same `- to:` item instead and you get one rule that only allows port 53 to
`api` pods. That blocks both the app traffic and DNS. **Rule of thumb:** count the top-level `-`
items under `egress`/`ingress`; that's the number of independent rules.

The same applies inside `from`/`to`: `namespaceSelector` and `podSelector` in the **same** item are
AND-ed ("these pods in those namespaces"); as two separate `-` items they're OR-ed.

### An Egress policy without a DNS rule silently breaks name resolution
Once `policyTypes` includes `Egress`, **all** egress that isn't explicitly allowed is denied,
including the pod's DNS queries to CoreDNS. The symptom (`could not resolve host`) looks like a
broken Service, but the policy is the cause. Almost every exam question that adds an egress policy
implicitly needs a port-53 rule. Add one by default.

A tighter version scopes DNS to CoreDNS instead of port 53 anywhere:
```yaml
- to:
  - namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: kube-system}
    podSelector:
      matchLabels: {k8s-app: kube-dns}
  ports:
  - port: 53
    protocol: UDP
  - port: 53
    protocol: TCP
```
Every namespace automatically carries the `kubernetes.io/metadata.name` label, and CoreDNS pods are
labelled `k8s-app: kube-dns`. Use this when the question says "only to DNS/CoreDNS". Otherwise the
simpler version has less to get wrong.

---

## Imperative flag gotchas

### `--annotations` doesn't split on commas, but `--labels` and `--env` do
```bash
k run x --image=nginx --labels="a=1,b=2"                       # two labels
k run x --image=nginx --annotations="a=1,b=2"                  # ONE annotation, value "1,b=2"
k run x --image=nginx --annotations="a=1" --annotations="b=2"  # correct
```
When in doubt, repeat the flag for each key, and check the generated YAML (`--dry-run=client -o yaml`) before applying.

### `--command` vs bare args after `--`
Without `--command`, everything after `--` becomes the container's `args:` (replacing the image's
`CMD` and keeping its `ENTRYPOINT`). With `--command`, the same tokens become `command:` (replacing
the `ENTRYPOINT`). See [drill 19.2](../drills/drill.md).

### `--dry-run` must be `--dry-run=client`
A bare `--dry-run` is deprecated. Forget the flag entirely and the object is really created, so the
YAML you export picks up `uid`, `resourceVersion`, `creationTimestamp` and a `status:` block. Use
`--dry-run=client -o yaml` every time.

---

## Exploring the cluster without leaving the terminal

### `kubectl explain` and `kubectl <command> -h`
These two are the most useful tools in the exam. They have their own page:
[kubectl-help.md](kubectl-help.md).

### `kubectl api-resources`: what exists and what it's called
```bash
k api-resources | grep -i network          # short names and API groups
k api-resources --namespaced=true -o name
k api-resources --api-group=networking.k8s.io
k get crd                                  # the "discover" half of "discover and use CRDs"
```

### `kubectl get all` doesn't get everything
It covers pods, services, deployments, replicasets, statefulsets, daemonsets, jobs and cronjobs,
but **not** ingresses, configmaps, secrets, PVCs, network policies, ServiceAccounts, roles and so
on.
```bash
k get all,ing,cm,secret,pvc,netpol,sa,role,rolebinding -n dev
# literally everything namespaced (paste avoids tr's trailing comma):
k get $(k api-resources --namespaced=true --verbs=list -o name | paste -sd,) -n dev
```

---

## kubectl debug

`kubectl run` creates a **new** pod. `kubectl debug` gives you tools *alongside* an existing one,
which matters when the app image is distroless and has no shell for `kubectl exec`. It has three
modes.

**1. Ephemeral container: attach to a live pod without restarting it**
```bash
k debug -it mypod --image=busybox --target=mycontainer -- sh
```
This adds a container to the running pod (shown under *Ephemeral Containers* in `describe`).
`--target` shares the target container's **process namespace**, so `ps` shows the app's processes,
but **not its filesystem**. To browse the target's files, go through procfs:
`ls /proc/<pid>/root/`, where `<pid>` is the app's process from `ps`. Ephemeral containers can't be
removed; they go away when the pod is deleted.

For example, against a pod running `registry.k8s.io/pause` (no shell, so `exec` fails):
```
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /pause        <- the target container's process
   20 root      0:00 sh            <- your debug shell
/ # ls /proc/1/root/               <- the target's filesystem
```

**2. Copy-to: clone the pod with changes**
```bash
k debug mypod -it --image=busybox --copy-to=mypod-debug --container=debugger -- sh
k debug mypod -it --copy-to=mypod-debug --container=app -- sh    # replace the app's command
```
This creates a **new** pod. Use it when the live pod won't let you change something: override the
command of a crash-looping container so it stays up long enough to look around, or swap images with
`--set-image=app=busybox`.

**3. Node: a privileged pod with the node's filesystem at `/host`**
```bash
k debug node/<node-name> -it --image=busybox
chroot /host   # inside, operate as if on the node
```
Use it for node-level problems (disk, kubelet, host networking) rather than container ones.

---

## Handy one-liners

```bash
# temporary pod for testing, deleted on exit
k run tmp --rm -it --restart=Never --image=busybox -- sh
k run tmp --rm -it --restart=Never --image=curlimages/curl -- curl -s http://svc.ns:80

# events for one object, newest last
k get events --field-selector involvedObject.name=<pod> --sort-by=.lastTimestamp

# logs from every container of every matching pod
k logs -l app=web --all-containers --prefix
k logs <pod> -c <container> --previous      # the crashed instance

# get pod names for a Deployment's pods
k get pods -l app=web -o name
POD=$(k get pods -l app=web -o jsonpath='{.items[0].metadata.name}')

# Deployment and Service in one go
k create deployment web --image=nginx --replicas=2 --port=80
k expose deployment web --port=80 --target-port=80

# rollouts
k rollout status deploy/web
k rollout history deploy/web
k rollout undo deploy/web --to-revision=1
k rollout restart deploy/web

# delete immediately (with export now="--force --grace-period=0")
k delete pod web --force --grace-period=0
```
