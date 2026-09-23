# CKAD task patterns

The question *shapes* that kept coming back across the practice exams I took, grouped by task type
rather than by question. On exam day the goal is to recognise the pattern within the first sentence
and reach for the right idiom.

For the reasoning behind many of the gotchas, see [exam-tips.md](exam-tips.md). To practise these
end to end on a real cluster, see [../scenarios/](../scenarios/).

**Contents**

- [Namespace and context](#namespace-and-context)
- [Deployments: imperative first, then YAML](#deployments-imperative-first-then-yaml)
- [Blue/green and canary](#bluegreen-and-canary)
- [ConfigMaps and Secrets into a Pod](#configmaps-and-secrets-into-a-pod)
- [ServiceAccount on a Deployment](#serviceaccount-on-a-deployment)
- [RBAC: Role + RoleBinding](#rbac-role--rolebinding)
- [Services: expose, create, NodePort](#services-expose-create-nodeport)
- [Ingress](#ingress)
- [NetworkPolicy](#networkpolicy)
- [Multi-container Pods: sidecar and init containers](#multi-container-pods-sidecar-and-init-containers)
- [Probes](#probes)
- [Resources](#resources)
- [Images: Dockerfile and podman](#images-dockerfile-and-podman)
- [Helm](#helm)
- [Kustomize overlay](#kustomize-overlay)
- [PersistentVolume and PersistentVolumeClaim](#persistentvolume-and-persistentvolumeclaim)
- [Updating a running Deployment](#updating-a-running-deployment)
- [Troubleshooting](#troubleshooting)
- [Writing output to an answer file](#writing-output-to-an-answer-file)
- [Recurring themes](#recurring-themes)

---

## Namespace and context
Nearly every question opens this way, so make it a reflex:
```bash
kubectl config use-context <context-from-the-question>
kubectl create ns dev                                   # only if the question says it doesn't exist
kubectl config set-context --current --namespace=dev
kubectl config view --minify | grep namespace           # verify
```

---

## Deployments: imperative first, then YAML
```bash
kubectl create deployment app-a --image=nginx --replicas=2 --port=80 $do > app-a.yaml
# edit app-a.yaml for anything the flags can't express (env, probes, volumes, resources...)
kubectl apply -f app-a.yaml
kubectl rollout status deployment app-a
```

Create and expose (`--port` sets the containerPort; `expose` creates a ClusterIP Service unless
you pass `--type`):
```bash
kubectl create deployment ml-model --image=ml-serving:latest --replicas=2 --port=8080
kubectl expose deployment ml-model --port=8080 --type=NodePort
```

Adding an env var in the generated YAML (or imperatively with
`kubectl set env deploy/ml-model MODEL_NAME=my_model`):
```yaml
env:
- name: MODEL_NAME
  value: "my_model"
```

**Gotchas I actually hit:**
- `kubectl get deployment app-a -n -production`: a stray `-` before the namespace turns it into a
  flag.
- Forgetting `--dry-run=client` means the object really gets created before you export it, so the
  YAML carries `uid`, `resourceVersion`, `creationTimestamp` and `status:`. Use `$do` every time.

---

## Blue/green and canary

**Blue/green: two Deployments, flip the Service selector.** `web-app` (blue) and `web-app-green`
have identical specs but a different label (e.g. `version: blue` / `version: green`). The Service's
`spec.selector` is switched from one to the other to cut traffic over in one step:
```bash
kubectl patch service web-app -p '{"spec":{"selector":{"app":"web-app","version":"green"}}}'
```

Cloning a Deployment manifest under a new name is quicker with `sed` than by hand. Check the result
before applying, because `sed` also rewrites any image name that contains the string:
```bash
sed 's/web-app/web-app-green/g' web-app.yaml > web-app-green.yaml
kubectl apply -f web-app-green.yaml
```

**Canary: a new Deployment sharing the Service's label.** The Service selects a label that *both*
versions carry, so traffic is split roughly by replica count rather than cut over:
```bash
kubectl create deployment api-v2-canary --image=vector/api:v2 --replicas=1 $do > canary.yaml
# edit canary.yaml: add `app: api` to spec.template.metadata.labels (and the selector)
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-canary-service
spec:
  type: NodePort
  selector:
    app: api          # matches pods from both api-v1 and api-v2-canary
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30000
```
With 4 v1 replicas and 1 canary replica, about 20% of requests hit v2. Check that both are behind the
Service with `kubectl get endpoints api-canary-service` or `kubectl get pods -l app=api`.

---

## ConfigMaps and Secrets into a Pod

Three idioms. Know all of them cold, and read the question carefully for which one it wants:

| Idiom | Use when the question says... |
|---|---|
| `envFrom: - configMapRef / secretRef` | "all keys as environment variables" (names unchanged) |
| `env[].valueFrom.configMapKeyRef / secretKeyRef` | specific keys, or a different env var name |
| `volumes[].configMap / secret` + `volumeMounts` | "mount", "as files", "at path /x" |

```bash
kubectl create configmap web-config --from-literal=port=8080 --from-literal=document_root=/var/www/html
kubectl create secret generic db-credentials --from-literal=username=db_user --from-literal=password='P@ssw0rd123'
kubectl create configmap app-config --from-file=app.properties   # key = file name
kubectl create configmap app-env --from-env-file=app.env          # one key per line in the file
```

```yaml
spec:
  containers:
  - name: web
    image: nginx
    envFrom:
    - configMapRef:
        name: web-config          # env var names = keys as-is: port, document_root
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:             # secretKeyRef, not configMapKeyRef: easy to mix up under pressure
          name: db-credentials
          key: password
    volumeMounts:
    - name: config
      mountPath: /etc/app-config  # each key becomes a file: /etc/app-config/app.properties
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```
Verify with `kubectl exec <pod> -- env` or `kubectl exec <pod> -- ls /etc/app-config`.

**Gotchas:**
- **Most of a running Pod's spec is immutable.** You can't add `env`, `envFrom` or volumes to a live
  Pod with `patch` or `apply`. Export it, edit, then recreate it:
  `kubectl get pod web -o yaml > web.yaml`, edit, then `kubectl replace --force -f web.yaml`. (A
  Deployment is different: edit the template and it rolls out new pods.)
- **`$` in a `--from-literal` value.** A value like `mysql://admin:Pa$$w0rd@db` gets mangled by shell
  expansion before `kubectl` sees it. Use single quotes, then check what was stored:
  `kubectl get secret db -o jsonpath='{.data.url}' | base64 -d`.
- **Mounting one file without hiding the directory.** Mounting a ConfigMap at `/etc/nginx/conf.d`
  replaces everything in that directory. To drop in a single file, use `subPath` (see the sidecar example below). A
  `subPath` mount doesn't receive ConfigMap updates.

---

## ServiceAccount on a Deployment
```bash
kubectl create sa restricted-sa
kubectl set serviceaccount deployment app-a restricted-sa    # imperative
```
```yaml
spec:
  template:
    spec:
      serviceAccountName: restricted-sa   # on the POD TEMPLATE spec, not the Deployment's spec
```

---

## RBAC: Role + RoleBinding
```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods,pods/log -n dev
kubectl create rolebinding pod-reader-binding --role=pod-reader \
  --serviceaccount=dev:restricted-sa -n dev
kubectl auth can-i list pods --as=system:serviceaccount:dev:restricted-sa -n dev     # yes
kubectl auth can-i delete pods --as=system:serviceaccount:dev:restricted-sa -n dev   # no
```
Watch whether the question also wants the RoleBinding. It's easy to stop one step short. Log access is
the sub-resource `pods/log`.

A RoleBinding can grant a Role to a ServiceAccount from **another** namespace. `roleRef` always
refers to a Role in the RoleBinding's own namespace; the subject carries its own namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: security      # the ServiceAccount's namespace, not the binding's
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Services: expose, create, NodePort
**`kubectl expose`** reads the selector from an existing object. It's the fastest route when the
question names a Deployment or Pod:
```bash
kubectl scale deployment backend --replicas=5
kubectl expose deployment backend --name=backend-service --port=8080 --target-port=8080   # ClusterIP
kubectl expose deployment backend --name=backend-np --port=80 --type=NodePort
```

**`kubectl create service`** builds a standalone Service. It has **no `--selector` flag**: the
selector is always `app=<service-name>`. If the pods carry a different label, create it with `$do`
and edit the selector, or patch it afterwards:
```bash
kubectl create service nodeport auth-service --tcp=8443:8080 --node-port=30443   # selector app=auth-service
kubectl create service clusterip db --tcp=5432:5432 $do > db-svc.yaml             # then edit the selector
```
`--tcp=<port>:<targetPort>` sets both ports at once.

**ClusterIP → NodePort** on an existing Service: `kubectl edit svc <name>`, change `type:` and
optionally add `nodePort:` (30000–32767) under the port. To test, curl `<node-ip>:<nodePort>` or use a
temporary pod.

---

## Ingress
Imperative (see `kubectl create ingress -h` for the rule syntax):
```bash
kubectl create ingress web-ingress --class=nginx \
  --rule="shop.example.com/=web-service:80" \
  --rule="shop.example.com/api*=api-service:8080"
```
`path` without `*` means `pathType: Exact`, and `path*` means `Prefix`. Or declaratively:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  tls:                               # optional; the Secret already exists in exam questions
  - hosts: [api.example.com]
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-api
            port:
              number: 443
```
Test through the controller with a `Host` header:
`curl -H 'Host: api.example.com' http://<ingress-address>/`.

---

## NetworkPolicy

**Allow ingress from a namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: app-v1
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend   # every namespace has this label automatically
    ports:
    - protocol: TCP
      port: 80
```
`namespaceSelector` matches labels on the **Namespace object**, not on pods. Check them with
`kubectl get ns --show-labels`. Older questions use a custom label such as `name: frontend`, which
has to exist on the namespace for the policy to match.

**Allow ingress from certain pods, leave egress open:**
```yaml
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: app
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - {}          # allow all egress
```
`egress: [{}]` is the idiom for "listed under policyTypes but not restricted". If you list `Egress`
with no rules, you've blocked all outbound traffic, DNS included.

**Default deny** for a namespace: `podSelector: {}` with `policyTypes: [Ingress]` (or both) and no
rules. For egress restrictions, remember the port-53 DNS rule; see
[exam-tips.md](exam-tips.md#networkpolicy-egress-gotchas).

---

## Multi-container Pods: sidecar and init containers

**Config file from a ConfigMap, mounted as a single file with `subPath`:**
```bash
kubectl create configmap haproxy-config --from-file=haproxy.cfg
```
```yaml
spec:
  containers:
  - name: poller
    image: poller-image:latest
    args: ["--url", "http://localhost:90"]   # containers in a pod share localhost
  - name: haproxy
    image: haproxy
    ports:
    - containerPort: 90
    volumeMounts:
    - name: haproxy-config
      mountPath: /usr/local/etc/haproxy/haproxy.cfg
      subPath: haproxy.cfg
  volumes:
  - name: haproxy-config
    configMap:
      name: haproxy-config
```

**Log-shipping sidecar: a shared `emptyDir`.** The app writes logs and the sidecar reads them. Here
two volumes do two jobs: `emptyDir` is the channel between the containers, and the ConfigMap is the
sidecar's own configuration.
```yaml
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: fluentd-config
    configMap:
      name: fluentd-config
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo \"$(date) log line\" >> /var/log/shared/app.log; sleep 1; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/shared
  - name: log-shipper
    image: fluent/fluentd:v1.16-1
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/shared
    - name: fluentd-config
      mountPath: /fluentd/etc
```
Watch the quoting when `$(date)` sits inside an already double-quoted `-c` string.

**Native sidecar (Kubernetes 1.29+):** an `initContainers` entry with `restartPolicy: Always` starts
before the app, keeps running beside it, and doesn't block Job completion. If a question says
"sidecar" on a recent cluster, this form may be what it expects:
```yaml
spec:
  initContainers:
  - name: log-tailer
    image: busybox
    restartPolicy: Always
    command: ["sh", "-c", "tail -F /var/log/app/app.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
```

**Init container pre-populating content:**
```yaml
spec:
  volumes:
  - name: content
    emptyDir: {}
  initContainers:
  - name: init-content
    image: busybox
    command: ["sh", "-c", "echo 'Hello, world!' > /usr/share/nginx/html/index.html"]   # absolute path!
    volumeMounts:
    - name: content
      mountPath: /usr/share/nginx/html
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: content
      mountPath: /usr/share/nginx/html
```
Both containers mount the **same** volume. The init container writes the file and exits, then the
main container starts and serves it.

---

## Probes
```yaml
containers:
- name: web
  image: my-app:1.0
  ports:
  - containerPort: 8080
  startupProbe:                # gives slow starters up to 30 × 5s = 150s before liveness kicks in
    httpGet: {path: /healthz, port: 8080}
    failureThreshold: 30
    periodSeconds: 5
  readinessProbe:              # failing = removed from Service endpoints, NOT restarted
    httpGet: {path: /ready, port: 8080}
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:               # failing = container restarted
    httpGet: {path: /healthz, port: 8080}
    periodSeconds: 15
```
Questions often use different paths for readiness and liveness. Don't copy one probe's `path` into
the other. Other probe types are `exec: {command: [...]}` and `tcpSocket: {port: 3306}`.

**Troubleshooting a failing probe:** `kubectl describe pod <pod>` shows the probe configuration and
`Warning Unhealthy` events at the bottom. Typical causes, in order:
1. The app doesn't listen on the probe's path or port (a typo, or the wrong port).
2. `initialDelaySeconds` is too short, or there's no startupProbe for a slow starter.
3. `timeoutSeconds` is too tight for an app that's slow under load.
4. The pod is starved of resources (CPU throttling, near its memory limit). Check `kubectl top pod`.

---

## Resources
```bash
kubectl set resources deployment web --requests=cpu=250m,memory=100Mi --limits=cpu=500m,memory=256Mi
```
```yaml
resources:
  requests:
    cpu: 250m
    memory: 100Mi
  limits:
    memory: 256Mi
```
If a namespace has a **ResourceQuota** on CPU or memory, every new pod *must* declare the quota'd
requests/limits or it's rejected. `kubectl describe quota` and the ReplicaSet's events
(`kubectl describe rs`) show why pods aren't being created. A **LimitRange** injects defaults into
pods that don't declare them.

---

## Images: Dockerfile and podman
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
WORKDIR /app
COPY my-app config.ini ./
CMD ["/app/my-app", "-c", "/app/config.ini"]
```
```bash
podman build -t my-app:1.0 .                       # in the directory with the Dockerfile
podman tag my-app:1.0 registry.example.com/my-app:1.0
podman push registry.example.com/my-app:1.0
podman run -d --name my-app -p 8080:80 my-app:1.0
podman logs my-app > /path/to/answer.log
podman save -o my-app.tar my-app:1.0               # "export the image to a tar"
```
See [exam-tips.md](exam-tips.md#dockerfile-instructions-memorise-them-theres-nothing-to-look-up) for
the instruction cheat sheet.

---

## Helm
```bash
helm list -A                                        # all namespaces
helm list -A --pending                              # stuck releases (Helm 3: -a shows all states)
helm show values bitnami/nginx | less               # what can be set
helm install my-app ./my-app-chart-0.1.0.tgz -n staging --create-namespace --set image.tag=v1.2.3
helm upgrade my-app bitnami/nginx -n staging --set replicaCount=3
helm get values my-app -n staging                   # what was set
helm history my-app -n staging
helm rollback my-app 1 -n staging
helm uninstall my-app -n staging
```
A chart can be a repo reference (`repo/chart`), a local directory, or a `.tgz`. `--set` uses dotted
paths for nested values (`image.tag`). `helm install` doesn't wait for pods unless you pass `--wait`,
so confirm with `kubectl get pods -n staging`.

---

## Kustomize overlay
```yaml
# overlay/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
namePrefix: prod-
patches:
- path: patch.yaml
```
```yaml
# overlay/patch.yaml: only the fields being overridden, plus enough to identify the object
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: web-app          # matched by name
        image: nginx:1.25
```
```bash
kubectl kustomize overlay/        # preview the rendered output
kubectl apply -k overlay/         # apply the directory, not the patch file
```
Older material uses `bases:` and `patchesStrategicMerge:`. They still work with a deprecation
warning, but `resources:` and `patches:` are the current fields.

---

## PersistentVolume and PersistentVolumeClaim

**Static provisioning (hostPath PV).** If the question has you prepare a directory on a node, it
wants a hand-written `hostPath` PV:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: exam            # must match the PVC's
  hostPath:
    path: /opt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Mi
  storageClassName: exam
```
The PVC binds to a PV with the same `storageClassName`, a compatible access mode and enough capacity.
If the question says "no storageClassName", set `storageClassName: ""` explicitly on **both**.
Omitting the field on the PVC lets the cluster's default StorageClass grab it instead.

**Dynamic provisioning.** If the question only gives a PVC with a `storageClassName` and never
mentions a node directory, don't write a PV: the StorageClass's provisioner creates one. A PVC
stuck in `Pending` usually means a StorageClass with a non-existent provisioner, or
`volumeBindingMode: WaitForFirstConsumer` waiting for a pod. `kubectl describe pvc` tells you which.

Mount it:
```yaml
spec:
  containers:
  - name: postgres
    image: postgres:16
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: task-pvc
```

---

## Updating a running Deployment

**Changing the image:**
```bash
kubectl set image deployment/api-service api=registry/api:v2     # <container-name>=<image>
kubectl rollout status deployment/api-service
```
`set image` is faster and safer than a hand-written patch. If you do patch, the container `name` must
match exactly. A strategic-merge patch with a wrong name *adds a second container* instead of
updating the first:
```bash
kubectl patch deployment api-service -p '{"spec":{"template":{"spec":{"containers":[{"name":"api","image":"registry/api:v2"}]}}}}'
```

**Rolling-update tuning:**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # or "25%": extra pods allowed above replicas
      maxUnavailable: 0    # or "25%": pods allowed below replicas
```
`maxUnavailable: 0` with `maxSurge >= 1` is the "never drop below full capacity" (zero-downtime)
answer. `type: Recreate` kills everything first. Use it when the question says old and new versions
must never run together.

**History and rollback:**
```bash
kubectl rollout history deployment/app
kubectl rollout history deployment/app --revision=2     # what changed in revision 2
kubectl rollout undo deployment/app                     # previous revision
kubectl rollout undo deployment/app --to-revision=1
kubectl annotate deployment/app kubernetes.io/change-cause="bump to v2"   # fills CHANGE-CAUSE
```
Once a bad image or config has rolled out, `rollout undo` is the expected fix, rather than
re-editing the old values by hand.

**Pod template vs Deployment metadata.** When a question says "add label X to the pods", edit
`spec.template.metadata.labels`, not the Deployment's own `metadata.labels`. A Service selects on the
**pod** labels.

---

## Troubleshooting

**"Clients can't reach the Service"**, in this order:
```bash
kubectl describe svc api-service          # selector, port, targetPort
kubectl get endpoints api-service         # empty = selector/label mismatch (the #1 cause)
kubectl get pods -l app=web --show-labels # READY column, actual labels
kubectl describe pod <pod>                # probes, events
kubectl get netpol                        # is something blocking it?
```

**"Something's broken, find it"**, when you don't know the namespace:
```bash
kubectl get pods -A | grep -v Running
kubectl get pods -A | grep -i web-app
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

**Common pod states and what they usually mean:**

| Status | Usual cause | Where to look |
|---|---|---|
| `Pending` | no node fits (resources, taints, nodeSelector), unbound PVC, quota | `describe pod` events |
| `ImagePullBackOff` / `ErrImagePull` | wrong image name or tag, private registry | `describe pod` events |
| `CrashLoopBackOff` | app exits: bad command, missing config, failing liveness probe | `logs --previous`, `describe` |
| `CreateContainerConfigError` | referenced ConfigMap/Secret or key doesn't exist | `describe pod` events |
| `OOMKilled` | memory limit too low | `describe pod` (Last State) |
| Running but `0/1` READY | readiness probe failing | `describe pod` events |

**Logs and events:**
```bash
kubectl logs -l app=my-app --all-containers --prefix     # every container of every matching pod
kubectl logs <pod> -c <container> --previous             # the crashed instance
kubectl get events --field-selector involvedObject.name=<pod>
kubectl top pods --sort-by=cpu                           # needs metrics-server
```

---

## Writing output to an answer file
Questions often ask you to save something to a specific path. Grading reads that file:
```bash
kubectl get pod web -o yaml > /opt/task1/pod.yaml
kubectl get pods -o jsonpath='{.items[*].metadata.name}' > /opt/task1/names.txt
kubectl get pod web -o custom-columns=NAME:.metadata.name,NS:.metadata.namespace --no-headers > /opt/task1/broken.txt
kubectl get events --field-selector involvedObject.name=web > /opt/task1/events.txt
kubectl logs web > /opt/task1/web.log
```
Use `-o yaml`/`-o json` for a full object, and jsonpath or custom-columns for specific fields. `cat`
the file afterwards. Scripts ("write a command that...") need the command itself in the file, not
its output: `echo 'kubectl get pod web -o jsonpath="{.status.phase}"' > /opt/task1/status.sh`.

---

## Recurring themes
- **Namespace first.** Virtually every question starts with a context and namespace switch.
- **Troubleshooting rewards a fixed order:** describe → endpoints → readiness → policy/DNS. `describe`
  and events answer most questions faster than logs do.
- **ConfigMap/Secret injection** (`envFrom` vs `valueFrom.*KeyRef` vs volume) comes up in several
  questions. Know all three.
- **Dry-run discipline.** Forgetting `--dry-run=client` before redirecting to a file creates the
  object for real. Use `$do`.
- **Pods are mostly immutable.** Env vars, volumes and most container fields can't be changed in
  place. Use `replace --force` for bare Pods; Deployments roll out a new template.
- **Deployment changes go through the pod template.** `set image`/`patch`/`edit` target
  `spec.template.spec`, and Service selectors match `spec.template.metadata.labels`.
- **Quote `--from-literal` values** that contain `$`, and verify with `jsonpath` + `base64 -d`.
- **A PV isn't always yours to write.** A node-prep step means a static `hostPath` PV. A bare PVC
  with a StorageClass means dynamic provisioning.
- **Broken rollouts get `rollout undo`**, not a manual re-edit.
- **Answer files must exist on disk** at exactly the path given.
