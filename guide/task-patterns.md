# CKAD task patterns

The question *shapes* that kept coming back across the practice exams I took, grouped by task type
rather than by question. On exam day the goal is to recognise the pattern within the first sentence
and reach for the right idiom.

> **Where this comes from.** This page is partly derivative. The task types, and several of the
> gotchas, come from working through practice exams, mainly KodeKloud's
> [Ultimate CKAD Mock Exam Series](https://learn.kodekloud.com/user/courses/ultimate-certified-kubernetes-application-developer-ckad-mock-exam-series)
> and MyExamCloud's [free CKAD practice tests](https://www.myexamcloud.com/onlineexam/ckad-free-practice-tests.course).
> The grouping, explanations and examples are my own; the examples use a fictional bike-rental
> company instead of the names in those questions. For the original questions, go to the sources.

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
ssh <vm-from-the-question>                              # each question has its own VM
kubectl create ns rentals                               # only if the question says it doesn't exist
kubectl config set-context --current --namespace=rentals
kubectl config view --minify | grep namespace           # verify
```

---

## Deployments: imperative first, then YAML
```bash
kubectl create deployment station-api --image=nginx --replicas=2 --port=80 --dry-run=client -o yaml > station-api.yaml
# edit station-api.yaml for anything the flags can't express (env, probes, volumes, resources...)
kubectl apply -f station-api.yaml
kubectl rollout status deployment station-api
```

Create and expose (`--port` sets the containerPort; `expose` creates a ClusterIP Service unless
you pass `--type`):
```bash
kubectl create deployment route-planner --image=registry.example.com/route-planner:2.1 --replicas=2 --port=9000
kubectl expose deployment route-planner --port=9000 --type=NodePort
```

Adding an env var in the generated YAML (or imperatively with
`kubectl set env deploy/route-planner PLANNER_REGION=eu-west`):
```yaml
env:
- name: PLANNER_REGION
  value: "eu-west"
```

**Gotchas I actually hit:**
- `kubectl get deployment station-api -n -rentals`: a stray `-` before the namespace turns it into
  a flag.
- Forgetting `--dry-run=client` means the object really gets created before you export it, so the
  YAML carries `uid`, `resourceVersion`, `creationTimestamp` and `status:`. Type
  `--dry-run=client -o yaml` every time.

---

## Blue/green and canary

**Blue/green: two Deployments, flip the Service selector.** `booking-ui` (blue) and
`booking-ui-green` have identical specs but a different label (`version: blue` / `version: green`).
Switching the Service's `spec.selector` from one to the other cuts all traffic over in one step:
```bash
kubectl patch service booking-ui -p '{"spec":{"selector":{"app":"booking-ui","version":"green"}}}'
```

Cloning a Deployment manifest under a new name is quicker with `sed` than by hand. Check the result
before applying, because `sed` also rewrites any image name that contains the string:
```bash
sed 's/booking-ui/booking-ui-green/g' booking-ui.yaml > booking-ui-green.yaml
kubectl apply -f booking-ui-green.yaml
```

**Canary: a new Deployment sharing the Service's label.** The Service selects a label that *both*
versions carry, so traffic is split roughly by replica count rather than cut over:
```bash
kubectl create deployment pricing-v2 --image=registry.example.com/pricing:2.0 --replicas=1 --dry-run=client -o yaml > pricing-v2.yaml
# edit pricing-v2.yaml: add `app: pricing` to spec.template.metadata.labels (and the selector)
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: pricing
spec:
  type: NodePort
  selector:
    app: pricing      # matches Pods from both pricing-v1 and pricing-v2
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31080
```
With 4 `pricing-v1` replicas and 1 `pricing-v2` replica, about 20% of requests hit v2. Check that
both are behind the Service with `kubectl get endpoints pricing` or `kubectl get pods -l app=pricing`.

---

## ConfigMaps and Secrets into a Pod

Three idioms. Know all of them cold, and read the question carefully for which one it wants:

| Idiom | Use when the question says... |
|---|---|
| `envFrom: - configMapRef / secretRef` | "all keys as environment variables" (names unchanged) |
| `env[].valueFrom.configMapKeyRef / secretKeyRef` | specific keys, or a different env var name |
| `volumes[].configMap / secret` + `volumeMounts` | "mount", "as files", "at path /x" |

```bash
kubectl create configmap station-limits --from-literal=MAX_RENTALS=3 --from-literal=CURRENCY=EUR
kubectl create secret generic fleet-db --from-literal=username=fleet --from-literal=password='not-a-real-pw'
kubectl create configmap rental-settings --from-file=rental.properties   # key = file name
kubectl create configmap rental-env --from-env-file=rental.env            # one key per line in the file
```

```yaml
spec:
  containers:
  - name: station-api
    image: nginx
    envFrom:
    - configMapRef:
        name: station-limits      # env var names = keys as-is: MAX_RENTALS, CURRENCY
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:             # secretKeyRef, not configMapKeyRef: easy to mix up under pressure
          name: fleet-db
          key: password
    volumeMounts:
    - name: settings
      mountPath: /etc/rentals     # each key becomes a file: /etc/rentals/rental.properties
      readOnly: true
  volumes:
  - name: settings
    configMap:
      name: rental-settings
```
Verify with `kubectl exec <pod> -- env` or `kubectl exec <pod> -- ls /etc/rentals`.

**Gotchas:**
- **Most of a running Pod's spec is immutable.** You can't add `env`, `envFrom` or volumes to a live
  Pod with `patch` or `apply`. Export it, edit, then recreate it:
  `kubectl get pod station-api -o yaml > station-api.yaml`, edit, then
  `kubectl replace --force -f station-api.yaml`. (A Deployment is different: edit the template and
  it rolls out new Pods.)
- **`$` in a `--from-literal` value.** A value like `postgres://fleet:Ch$in@fleet-db` gets mangled
  by shell expansion before `kubectl` sees it. Use single quotes, then check what was stored:
  `kubectl get secret fleet-db-url -o jsonpath='{.data.url}' | base64 -d`.
- **Mounting one file without hiding the directory.** Mounting a ConfigMap at `/etc/nginx/conf.d`
  replaces everything in that directory. To drop in a single file, use `subPath` (see the sidecar
  example below). A `subPath` mount doesn't receive ConfigMap updates.

---

## ServiceAccount on a Deployment
```bash
kubectl create sa fleet-reader
kubectl set serviceaccount deployment station-api fleet-reader    # imperative
```
```yaml
spec:
  template:
    spec:
      serviceAccountName: fleet-reader   # on the POD TEMPLATE spec, not the Deployment's spec
```

---

## RBAC: Role + RoleBinding
```bash
kubectl create role pod-viewer --verb=get,list,watch --resource=pods,pods/log -n fleet
kubectl create rolebinding pod-viewer-binding --role=pod-viewer \
  --serviceaccount=fleet:fleet-reader -n fleet
kubectl auth can-i list pods --as=system:serviceaccount:fleet:fleet-reader -n fleet     # yes
kubectl auth can-i delete pods --as=system:serviceaccount:fleet:fleet-reader -n fleet   # no
```
Watch whether the question also wants the RoleBinding. It's easy to stop one step short. Log access
is the sub-resource `pods/log`.

A RoleBinding can grant a Role to a ServiceAccount from **another** namespace. `roleRef` always
refers to a Role in the RoleBinding's own namespace; the subject carries its own namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-viewer-binding
  namespace: fleet
subjects:
- kind: ServiceAccount
  name: dashboard
  namespace: monitoring    # the ServiceAccount's namespace, not the binding's
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

---

## Services: expose, create, NodePort
**`kubectl expose`** reads the selector from an existing object. It's the fastest route when the
question names a Deployment or Pod:
```bash
kubectl scale deployment dispatch --replicas=5
kubectl expose deployment dispatch --name=dispatch-svc --port=80 --target-port=8080   # ClusterIP
kubectl expose deployment dispatch --name=dispatch-np --port=80 --type=NodePort
```

**`kubectl create service`** builds a standalone Service. It has **no `--selector` flag**: the
selector is always `app=<service-name>`. If the Pods carry a different label, create it with
`--dry-run=client -o yaml` and edit the selector, or patch it afterwards:
```bash
kubectl create service nodeport payments --tcp=9443:8443 --node-port=31443   # selector app=payments
kubectl create service clusterip ledger-db --tcp=5432:5432 --dry-run=client -o yaml > ledger-db-svc.yaml   # then edit the selector
```
`--tcp=<port>:<targetPort>` sets both ports at once.

**ClusterIP → NodePort** on an existing Service: `kubectl edit svc <name>`, change `type:` and
optionally add `nodePort:` (30000–32767) under the port. To test, curl `<node-ip>:<nodePort>` or use a
temporary Pod.

---

## Ingress
Imperative (see `kubectl create ingress -h` for the rule syntax):
```bash
kubectl create ingress rentals --class=nginx \
  --rule="rentals.example.com/=booking-ui:80" \
  --rule="rentals.example.com/prices*=pricing:80"
```
`path` without `*` means `pathType: Exact`, and `path*` means `Prefix`. Or declaratively, here with
TLS:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payments
spec:
  ingressClassName: nginx
  tls:                               # optional; the Secret already exists in exam questions
  - hosts: [pay.rentals.example.com]
    secretName: pay-tls
  rules:
  - host: pay.rentals.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: payments
            port:
              number: 9443
```
Test through the controller with a `Host` header:
`curl -H 'Host: rentals.example.com' http://<ingress-address>/`.

---

## NetworkPolicy

**Allow ingress from a namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ledger-from-billing
spec:
  podSelector:
    matchLabels:
      app: ledger
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: billing    # every namespace has this label automatically
    ports:
    - protocol: TCP
      port: 8080
```
`namespaceSelector` matches labels on the **Namespace object**, not on Pods. Check them with
`kubectl get ns --show-labels`. Some questions use a custom namespace label instead (say
`team: billing`), which has to exist on the namespace for the policy to match.

**Allow ingress from certain Pods, leave egress open:**
```yaml
spec:
  podSelector:
    matchLabels:
      app: fleet-db
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 5432
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

**A config file from a ConfigMap, mounted as a single file with `subPath`,** next to a sidecar that
talks to the main container over `localhost`:
```bash
kubectl create configmap station-nginx --from-file=stations.conf
```
```yaml
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-extra
      mountPath: /etc/nginx/conf.d/stations.conf   # just this file; default.conf stays in place
      subPath: stations.conf
  - name: health-check
    image: busybox
    command: ["sh", "-c", "while true; do wget -qO- http://localhost/ >/dev/null && echo ok; sleep 30; done"]
  volumes:
  - name: nginx-extra
    configMap:
      name: station-nginx
```
Containers in a Pod share a network namespace, so `localhost` reaches the other container.

**Log-forwarding sidecar: a shared `emptyDir`.** The app writes logs and the sidecar reads them. Here
two volumes do two jobs: `emptyDir` is the channel between the containers, and the ConfigMap is the
sidecar's own configuration.
```yaml
spec:
  volumes:
  - name: rental-logs
    emptyDir: {}
  - name: forwarder-config
    configMap:
      name: fluent-bit-config
  containers:
  - name: booking-ui
    image: busybox
    command: ["sh", "-c", "while true; do echo \"$(date) booking created\" >> /var/log/rentals/bookings.log; sleep 2; done"]
    volumeMounts:
    - name: rental-logs
      mountPath: /var/log/rentals
  - name: log-forwarder
    image: fluent/fluent-bit:3.0
    volumeMounts:
    - name: rental-logs
      mountPath: /var/log/rentals
      readOnly: true
    - name: forwarder-config
      mountPath: /fluent-bit/etc
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
    command: ["sh", "-c", "tail -F /var/log/rentals/bookings.log"]
    volumeMounts:
    - name: rental-logs
      mountPath: /var/log/rentals
```

**Init container preparing content:**
```yaml
spec:
  volumes:
  - name: status-page
    emptyDir: {}
  initContainers:
  - name: render-status
    image: busybox
    command: ["sh", "-c", "echo 'All stations online' > /usr/share/nginx/html/status.html"]   # absolute path!
    volumeMounts:
    - name: status-page
      mountPath: /usr/share/nginx/html
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: status-page
      mountPath: /usr/share/nginx/html
```
Both containers mount the **same** volume. The init container writes the file and exits, then the
main container starts and serves it.

---

## Probes
```yaml
containers:
- name: route-planner
  image: registry.example.com/route-planner:2.1
  ports:
  - containerPort: 9000
  startupProbe:                # gives slow starters up to 30 × 5s = 150s before liveness kicks in
    httpGet: {path: /healthz, port: 9000}
    failureThreshold: 30
    periodSeconds: 5
  readinessProbe:              # failing = removed from Service endpoints, NOT restarted
    httpGet: {path: /ready, port: 9000}
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:               # failing = container restarted
    httpGet: {path: /healthz, port: 9000}
    periodSeconds: 15
```
Readiness and liveness often check different paths. Don't copy one probe's `path` into the other.
Other probe types are `exec: {command: [...]}` and `tcpSocket: {port: 5432}`.

**Troubleshooting a failing probe:** `kubectl describe pod <pod>` shows the probe configuration and
`Warning Unhealthy` events at the bottom. Typical causes, in order:
1. The app doesn't listen on the probe's path or port (a typo, or the wrong port).
2. `initialDelaySeconds` is too short, or there's no startupProbe for a slow starter.
3. `timeoutSeconds` is too tight for an app that's slow under load.
4. The Pod is starved of resources (CPU throttling, near its memory limit). Check `kubectl top pod`.

---

## Resources
```bash
kubectl set resources deployment route-planner --requests=cpu=300m,memory=192Mi --limits=memory=384Mi
```
```yaml
resources:
  requests:
    cpu: 300m
    memory: 192Mi
  limits:
    memory: 384Mi
```
If a namespace has a **ResourceQuota** on CPU or memory, every new Pod *must* declare the quota'd
requests/limits or it's rejected. `kubectl describe quota` and the ReplicaSet's events
(`kubectl describe rs`) show why Pods aren't being created. A **LimitRange** injects defaults into
Pods that don't declare them.

---

## Images: Dockerfile and podman
```dockerfile
FROM python:3.12-slim
WORKDIR /srv
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
```
```bash
podman build -t station-api:1.3 .                          # in the directory with the Dockerfile
podman tag station-api:1.3 registry.example.com/station-api:1.3
podman push registry.example.com/station-api:1.3
podman run -d --name station-api -p 8080:8080 station-api:1.3
podman logs station-api > /path/to/answer.log
podman save -o station-api.tar station-api:1.3             # "export the image to a tar"
```
See [exam-tips.md](exam-tips.md#dockerfile-instructions-memorise-them-theres-nothing-to-look-up) for
the instruction cheat sheet.

---

## Helm
```bash
helm list -A                                            # all namespaces
helm list -A --pending                                  # stuck releases (Helm 3: -a shows all states)
helm show values bitnami/nginx | less                   # what can be set
helm install rentals-web ./rentals-web-2.4.0.tgz -n preprod --create-namespace --set image.tag=2.4.1
helm upgrade rentals-web ./rentals-web-2.4.0.tgz -n preprod --reuse-values --set replicaCount=3
helm get values rentals-web -n preprod                  # what was set
helm history rentals-web -n preprod
helm rollback rentals-web 1 -n preprod
helm uninstall rentals-web -n preprod
```
A chart can be a repo reference (`repo/chart`), a local directory, or a `.tgz`. `--set` uses dotted
paths for nested values (`image.tag`), and without `--reuse-values` an upgrade drops the values you
set earlier. `helm install` doesn't wait for Pods unless you pass `--wait`, so confirm with
`kubectl get pods -n preprod`.

---

## Kustomize overlay
```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: prod-
patches:
- path: scale-and-pin.yaml
```
```yaml
# overlays/prod/scale-and-pin.yaml: only the fields being overridden, plus enough to identify the object
apiVersion: apps/v1
kind: Deployment
metadata:
  name: station-api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: station-api      # matched by name
        image: nginx:1.27
```
```bash
kubectl kustomize overlays/prod/        # preview the rendered output
kubectl apply -k overlays/prod/         # apply the directory, not the patch file
```
Older material uses `bases:` and `patchesStrategicMerge:`. They still work with a deprecation
warning, but `resources:` and `patches:` are the current fields. More examples:
[../examples/kustomize/](../examples/kustomize/).

---

## PersistentVolume and PersistentVolumeClaim

**Static provisioning (hostPath PV).** If the question has you prepare a directory on a node, it
wants a hand-written `hostPath` PV:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ride-history-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual            # must match the PVC's
  hostPath:
    path: /srv/ride-history
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ride-history-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 200Mi
  storageClassName: manual
```
The PVC binds to a PV with the same `storageClassName`, a compatible access mode and enough capacity.
If the question says "no storageClassName", set `storageClassName: ""` explicitly on **both**.
Omitting the field on the PVC lets the cluster's default StorageClass grab it instead.

**Dynamic provisioning.** If the question only gives a PVC with a `storageClassName` and never
mentions a node directory, don't write a PV: the StorageClass's provisioner creates one. A PVC
stuck in `Pending` usually means a StorageClass with a non-existent provisioner, or
`volumeBindingMode: WaitForFirstConsumer` waiting for a Pod. `kubectl describe pvc` tells you which.

Mount it:
```yaml
spec:
  containers:
  - name: archiver
    image: busybox
    command: ["sh", "-c", "date >> /data/runs.log; sleep 3600"]
    volumeMounts:
    - name: history
      mountPath: /data
  volumes:
  - name: history
    persistentVolumeClaim:
      claimName: ride-history-pvc
```

---

## Updating a running Deployment

**Changing the image:**
```bash
kubectl set image deployment/pricing pricing=registry.example.com/pricing:2.1   # <container-name>=<image>
kubectl rollout status deployment/pricing
```
`set image` is faster and safer than a hand-written patch. If you do patch, the container `name` must
match exactly. A strategic-merge patch with a wrong name *adds a second container* instead of
updating the first:
```bash
kubectl patch deployment pricing -p '{"spec":{"template":{"spec":{"containers":[{"name":"pricing","image":"registry.example.com/pricing:2.1"}]}}}}'
```

**Rolling-update tuning:**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # or "25%": extra Pods allowed above replicas
      maxUnavailable: 0    # or "25%": Pods allowed below replicas
```
`maxUnavailable: 0` with `maxSurge >= 1` is the "never drop below full capacity" (zero-downtime)
answer. `type: Recreate` kills everything first. Use it when the question says old and new versions
must never run together.

**History and rollback:**
```bash
kubectl rollout history deployment/pricing
kubectl rollout history deployment/pricing --revision=2     # what changed in revision 2
kubectl rollout undo deployment/pricing                     # previous revision
kubectl rollout undo deployment/pricing --to-revision=1
kubectl annotate deployment/pricing kubernetes.io/change-cause="pricing 2.1"   # fills CHANGE-CAUSE
```
Once a bad image or config has rolled out, `rollout undo` is the expected fix, rather than
re-editing the old values by hand.

**Pod template vs Deployment metadata.** When a question says "add label X to the Pods", edit
`spec.template.metadata.labels`, not the Deployment's own `metadata.labels`. A Service selects on the
**Pod** labels.

---

## Troubleshooting

**"Clients can't reach the Service"**, in this order:
```bash
kubectl describe svc dispatch-svc             # selector, port, targetPort
kubectl get endpoints dispatch-svc            # empty = selector/label mismatch (the #1 cause)
kubectl get pods -l app=dispatch --show-labels # READY column, actual labels
kubectl describe pod <pod>                    # probes, events
kubectl get netpol                            # is something blocking it?
```

**"Something's broken, find it"**, when you don't know the namespace:
```bash
kubectl get pods -A | grep -v Running
kubectl get pods -A | grep -i booking
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

**Common Pod states and what they usually mean:**

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
kubectl logs -l app=booking-ui --all-containers --prefix   # every container of every matching Pod
kubectl logs <pod> -c <container> --previous               # the crashed instance
kubectl get events --field-selector involvedObject.name=<pod>
kubectl top pods --sort-by=cpu                             # needs metrics-server
```

---

## Writing output to an answer file
Questions often ask you to save something to a specific path. Grading reads that file:
```bash
kubectl get pod station-api -o yaml > /opt/task1/pod.yaml
kubectl get pods -o jsonpath='{.items[*].metadata.name}' > /opt/task1/names.txt
kubectl get pod station-api -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName --no-headers > /opt/task1/placement.txt
kubectl get events --field-selector involvedObject.name=station-api > /opt/task1/events.txt
kubectl logs station-api > /opt/task1/station-api.log
```
Use `-o yaml`/`-o json` for a full object, and jsonpath or custom-columns for specific fields. `cat`
the file afterwards. Scripts ("write a command that...") need the command itself in the file, not
its output: `echo 'kubectl get pod station-api -o jsonpath="{.status.phase}"' > /opt/task1/status.sh`.

---

## Recurring themes
- **Namespace first.** Virtually every question starts with a namespace switch on its VM.
- **Troubleshooting rewards a fixed order:** describe → endpoints → readiness → policy/DNS. `describe`
  and events answer most questions faster than logs do.
- **ConfigMap/Secret injection** (`envFrom` vs `valueFrom.*KeyRef` vs volume) comes up in several
  questions. Know all three.
- **Dry-run discipline.** Forgetting `--dry-run=client` before redirecting to a file creates the
  object for real.
- **Pods are mostly immutable.** Env vars, volumes and most container fields can't be changed in
  place. Use `replace --force` for bare Pods; Deployments roll out a new template.
- **Deployment changes go through the Pod template.** `set image`/`patch`/`edit` target
  `spec.template.spec`, and Service selectors match `spec.template.metadata.labels`.
- **Quote `--from-literal` values** that contain `$`, and verify with `jsonpath` + `base64 -d`.
- **A PV isn't always yours to write.** A node-prep step means a static `hostPath` PV. A bare PVC
  with a StorageClass means dynamic provisioning.
- **Broken rollouts get `rollout undo`**, not a manual re-edit.
- **Answer files must exist on disk** at exactly the path given.
