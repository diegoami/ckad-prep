# Setting up a local practice cluster

Everything in [drills/](../drills/), [scenarios/](../scenarios/) and [mock-exams/](../mock-exams/)
runs against a local [kind](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker) cluster. I used one
under WSL2 on Windows, but the same steps work on Linux and macOS.

The scenarios assume the cluster is called **`ckad`**, so the single node is named
`ckad-control-plane` and the kubectl context is `kind-ckad`.

## 1. Install the tools

Follow each project's own install page; versions move faster than this README does.

| Tool | Why | Install |
|---|---|---|
| Docker (or Podman) | kind runs its nodes as containers; image-build questions | [docs.docker.com](https://docs.docker.com/engine/install/) / [podman.io](https://podman.io/docs/installation) |
| kind | the cluster | [kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| kubectl | obviously | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| helm | Helm questions | [helm.sh](https://helm.sh/docs/intro/install/) |

Try to match the Kubernetes minor version the exam currently uses (it's listed on the
[CKAD curriculum](https://github.com/cncf/curriculum)); pick the node image with `--image` if your
kind release defaults to something else.

## 2. Create the cluster with ingress-ready port mappings

kind doesn't map host ports 80/443 into the node by default, so an ingress controller isn't reachable
from your machine unless the cluster is created with `extraPortMappings`. Save this as
`kind-ckad.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ckad
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```bash
kind create cluster --config kind-ckad.yaml
kubectl cluster-info --context kind-ckad
```

To start over at any point: `kind delete cluster --name ckad`, which wipes everything on it.

A second cluster is useful for practising context switches:
`kind create cluster --name ckad2`, then `kubectl config get-contexts`.

## 3. Ingress controller (ingress-nginx)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.hostPort.enabled=true \
  --set controller.service.type=NodePort \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.networkPolicyEnabled=false \
  --set controller.terminationGracePeriodSeconds=0

kubectl wait -n ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s
```

- `controller.hostPort.enabled=true` binds the controller directly to the node's ports 80/443,
  which the cluster config above maps through to your machine.
- `controller.service.type=NodePort`: kind has no cloud LoadBalancer.

The controller is now reachable at `http://localhost`. To test an Ingress with
`ingressClassName: nginx`:
```bash
curl -H 'Host: shop.example.com' http://localhost/
```

**Troubleshooting:**
- *The controller pod stays `Pending`*: the node lacks the `ingress-ready=true` label (you created
  the cluster without the config above). Fix it with
  `kubectl label node ckad-control-plane ingress-ready=true`.
- *No port mappings*: use `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80`
  and `curl -H 'Host: ...' localhost:8080`. That still goes through nginx and the Ingress rules.
- *No `ingressClassName` on your Ingress*: that only works when an IngressClass is marked default:
  `kubectl annotate ingressclass nginx ingressclass.kubernetes.io/is-default-class=true`.
- `nginx.ingress.kubernetes.io/rewrite-target: /` strips the matched path before forwarding, so a
  request to `/mypath` reaches the pod as `/`.

## 4. metrics-server (for `kubectl top` and HPA)

A bare kind cluster doesn't ship metrics-server, so `kubectl top` fails with *Metrics API not
available*. Install it:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
It starts but never becomes `Ready`, because it validates the kubelet's TLS certificate and kind's
kubelet certificates are self-signed. Add `--kubelet-insecure-tls`, either with
`kubectl edit deployment metrics-server -n kube-system` (append `- --kubelet-insecure-tls` to the
container's `args:`) or with a JSON patch:
```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl rollout status deployment metrics-server -n kube-system
```
Give it about 15 seconds after the rollout before `kubectl top pod` returns data. The exam cluster
already has metrics-server; this is a lab-only fix.

## 5. NetworkPolicy enforcement

kind's default CNI (kindnet) **didn't enforce NetworkPolicy in older releases** (kind v0.23 and
earlier): policies apply without error and have no effect, which looks like success. Recent
releases enforce it (confirmed on kind v0.31 with Kubernetes v1.35). Check yours with a deny-all
policy before trusting a result:

```bash
kubectl create ns np-test
kubectl -n np-test run web --image=nginx --port=80 --expose
kubectl -n np-test apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all}
spec: {podSelector: {}, policyTypes: [Ingress]}
EOF
kubectl -n np-test run tmp --rm -it --restart=Never --image=busybox -- wget -qO- -T 3 http://web
# enforced: "download timed out"; not enforced: the nginx welcome page
kubectl delete ns np-test
```

If it isn't enforced, create a separate cluster without the default CNI and install Calico:
```yaml
# add to the kind config
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
```
Then apply the Calico manifest from the
[Calico quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind). Nodes
stay `NotReady` until Calico is running.

## 6. Default StorageClass

kind ships a default StorageClass called `standard` (the local-path provisioner). That's
convenient, but it means a PVC **without** `storageClassName` gets dynamically provisioned instead
of binding to the PV you wrote by hand. When a question says "no storage class", set
`storageClassName: ""` explicitly on both the PV and the PVC.

## 7. Local image registry (optional)

For the image build/push scenarios, run a throwaway registry:
```bash
docker run -d --name registry -p 5000:5000 registry:2
podman push --tls-verify=false localhost:5000/my-app:1.0     # or docker push localhost:5000/my-app:1.0
docker rm -f registry                                        # when done
```

## WSL notes

- Run Docker Engine inside WSL, or Docker Desktop with WSL integration. Either works with kind.
- Ports mapped by kind (80/443 above) are reachable from Windows at `localhost` too.
- If `kubectl` can't reach the cluster after a Windows restart, start Docker again; kind's node
  containers restart with it.
