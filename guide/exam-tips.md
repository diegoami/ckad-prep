# CKAD exam tips

Practical things I learned preparing for (and passing) the CKAD that the curriculum doesn't tell you.
Most of it is about the exam environment, what the curriculum really expects, and the gotchas that
cost the most time under pressure.

For recurring question *shapes* (blue/green, ConfigMap injection, PV/PVC, and so on), see
[task-patterns.md](task-patterns.md). For drilling individual commands, see
[../drills/drill.md](../drills/drill.md).

**Contents**

- [The exam environment](#the-exam-environment)
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

## The exam environment

### Documentation allowed during the exam
You get one browser with access to the official docs only:
- `kubernetes.io/docs`: the main reference, and it covers almost everything
- `kubernetes.io/blog`
- `helm.sh/docs`: the Helm CLI reference
- task-specific docs linked in a question's *Quick Reference* box (for example `docs.podman.io` on
  an image-building question)

No GitHub, no Stack Overflow, no tutorial sites. Practise finding things on `kubernetes.io/docs`
quickly. The search works, but knowing that "Configure Liveness, Readiness and Startup Probes" lives
under *Tasks → Configure Pods and Containers* saves minutes. The list changes occasionally, so check
the Linux Foundation's
[resources allowed during exams](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed)
page before you sit the exam.

### Always switch context (and namespace) first
Each question names the cluster context (or host to `ssh` into) it expects. Run that command
before anything else, every time, even when you think you're already there. Then set the namespace
so you don't have to type `-n` on every command:

```bash
k config set-context --current --namespace=<namespace>
k config view --minify | grep namespace   # verify
```

This is the most-used command on the exam. Forgetting it is the classic way to do a question
perfectly in the wrong place.

### Shell setup: do this in the first minute
```bash
alias k=kubectl                          # usually preconfigured, check with `type k`
source <(kubectl completion bash)
complete -o default -F __start_kubectl k # tab completion for the alias too
export do="--dry-run=client -o yaml"     # k run nginx --image=nginx $do > pod.yaml
export now="--force --grace-period=0"    # k delete pod nginx $now
```

`source` has to come before `complete`, because `__start_kubectl` is defined by the source line.
Without the `complete` line, tab completion on `k` does nothing.

A namespace-switch function saves retyping `config set-context` on every question:
```bash
kns() { kubectl config set-context --current --namespace="$1"; }
```

Save shell history after every command rather than only on a clean exit. A crashed terminal or a
hung `kubectl exec` then doesn't lose the commands you already ran:
```bash
shopt -s histappend
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
```
`history -a` appends new commands to `~/.bash_history` after every prompt; `history -c; history -r`
reloads from the file so several open terminals stay in sync.

### `sudo -i` changes `$HOME`
Reported by exam takers: switching to a root shell with `sudo -i` changes `$HOME` (and possibly the
working directory), so a file written before or after the switch can look "missing". It's still on
disk, under the other user's home. Before assuming a file was never written, check which shell
you're in; `find / -name <filename> 2>/dev/null` settles it.

### vim settings for YAML
The exam terminal's `vim` has no `.vimrc` of yours. If you'll edit YAML by hand, set one up first:
```bash
cat >> ~/.vimrc <<'EOF'
set number
set expandtab
set tabstop=2
set shiftwidth=2
set pastetoggle=<F5>
EOF
```
- **Real tabs break YAML.** Without `expandtab`, pressing Tab (or indenting with `>>` or visual `>`)
  inserts a literal tab, which `kubectl apply` rejects. To find stray tabs in an existing manifest,
  use `:set list` (tabs show as `^I`), then fix them with `:%s/\t/  /g`.
- **Pasting YAML from the browser without paste mode cascades the indentation.** Vim's autoindent
  reacts to every pasted line, so the nesting grows deeper with each one. Run `:set paste` (or press
  `<F5>` with the mapping above) before pasting and `:set nopaste` afterwards.
- **`Ctrl-W` closes the browser tab, not the vim split**, in browser-based exam terminals. Use
  `:wincmd w` (or `:wincmd h/j/k/l`) to move between splits, or use tmux (below).
- **Indent a block in one go:** `V`, extend with `j`, then `>`. `3>>` indents the next 3 lines
  without entering visual mode.

### tmux: split panes without the Ctrl-W conflict
tmux's prefix is `Ctrl+b`, so it avoids the browser swallowing shortcuts. Use `Ctrl+b %` for a
vertical split, `Ctrl+b "` for a horizontal one, `Ctrl+b` + arrow keys to move between panes and
`Ctrl+b z` to zoom a pane in and out. Sessions also survive a dropped connection: create one with
`tmux new -s ckad`, detach with `Ctrl+b d`, reattach with `tmux attach -t ckad`.

A minimal `~/.tmux.conf` (load it with `tmux source-file ~/.tmux.conf`):
```
set -sg escape-time 0      # no lag when pressing Esc in vim inside tmux
set -g history-limit 10000
set -g mouse off           # the browser handles copy/paste more predictably
set -g status-keys vi
set -g mode-keys vi
```

### Cluster vs context: there's no `use-cluster`
A **cluster** entry (`kubectl config get-clusters`) is only connection info: API server URL and CA
certificate. It has no identity and no namespace, and no command targets it directly. A **context**
bundles a cluster, a user and a default namespace into the unit kubectl actually works against.
"Switch to cluster X" always means switching context.

```bash
k config get-contexts                     # the * marks the active one
k get nodes --context other-ctx           # one-off, doesn't change your default
k config use-context other-ctx            # persisted for every later command
```

### Save answers to the exact file path asked for
Many questions say "write the output to `/some/path/file`". Grading reads that file. A correct
answer that only appeared in your terminal scores nothing. Redirect with `>` to the exact path, then
`cat` it to check.

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
When in doubt, repeat the flag for each key, and check the generated YAML (`$do`) before applying.

### `--command` vs bare args after `--`
Without `--command`, everything after `--` becomes the container's `args:` (replacing the image's
`CMD` and keeping its `ENTRYPOINT`). With `--command`, the same tokens become `command:` (replacing
the `ENTRYPOINT`). See [drill 19.2](../drills/drill.md).

### `--dry-run` must be `--dry-run=client`
A bare `--dry-run` is deprecated. Forget the flag entirely and the object is really created, so the
YAML you export picks up `uid`, `resourceVersion`, `creationTimestamp` and a `status:` block. Use
the `$do` variable every time.

---

## Exploring the cluster without leaving the terminal

### `kubectl explain`: use the full path and `--recursive`
```bash
k explain pod.spec.containers.livenessProbe
k explain deploy.spec.strategy.rollingUpdate
k explain pod.spec --recursive | grep -i -A3 capabilities   # "where does this field live?"
```
It's faster than clicking through the docs, and it always matches the cluster's actual API
version.

### `kubectl <verb> -h`: for command flags and examples
`explain` describes the **resource schema**; `-h` describes **the command**, with copy-pasteable
examples at the bottom. `explain` won't tell you what `--to-revision` does, and `-h` won't tell you
which fields a probe accepts.
```bash
k create deployment -h
k rollout undo -h
k create ingress -h      # the --rule syntax is hard to remember
k create cronjob -h
```

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
k delete pod web $now
```
