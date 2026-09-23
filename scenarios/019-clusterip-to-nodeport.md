# 019 — Convert a ClusterIP Service to NodePort, test via node IP

**Domain:** Services and Networking · **Difficulty:** Easy

## Task

> In namespace `jade`, the Deployment `status-page` runs one `httpd:2.4-alpine` Pod, exposed
> inside the cluster by the ClusterIP Service `status-page-svc` on port `8080`. An external
> monitoring probe needs to reach it without going through an Ingress. Change the Service to type
> NodePort with the fixed node port `30180`. Test it by sending a request straight to a node's
> internal IP on that port. Which nodes answer on `30180`, and on which node is the Pod running?

## Documentation

What to look up: **Service** — the `NodePort` type specifically.
- <https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport> — the
  30000-32767 valid port range and how it relates to the ClusterIP a NodePort Service still gets.

## Setup

```bash
kubectl create ns jade
kubectl -n jade create deployment status-page --image=httpd:2.4-alpine --port=80
kubectl -n jade rollout status deployment status-page --timeout=60s

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: status-page-svc
  namespace: jade
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 80
  selector:
    app: status-page
EOF
```

**Single-node caveat:** on a multi-node cluster you'd see that a NodePort Service answers on
**every** node, not just the one running the Pod (that's what NodePort means). A default `kind`
cluster has a single node (`ckad-control-plane`), so there's only one node IP to test against.
The steps are the same; you just can't see a node that isn't running the Pod answer as well.

## Solution

```bash
# baseline: works in-cluster as a ClusterIP Service
kubectl -n jade run tmp --restart=Never --rm -i --image=nginx:alpine -- \
  curl -s -m 5 status-page-svc:8080
# ... <title>It works! Apache httpd</title> ...

# change the type and pin the nodePort in one patch
kubectl -n jade patch service status-page-svc -p \
  '{"spec":{"type":"NodePort","ports":[{"name":"http","port":8080,"targetPort":80,"nodePort":30180}]}}'
# (or: kubectl -n jade edit svc status-page-svc — set type: NodePort and add nodePort: 30180)

kubectl -n jade get svc status-page-svc
# TYPE NodePort, PORT(S) 8080:30180/TCP

kubectl get nodes -o wide
# the INTERNAL-IP column has the address to hit directly

NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl -s -m 5 "http://$NODE_IP:30180"
# ... <title>It works! Apache httpd</title> ...

kubectl -n jade get pods -o wide
# the NODE column shows where the Pod landed; the Service answers on every node regardless
```

Answer to the two questions: every node answers on `30180`, and the Pod runs on whichever node
the NODE column shows (`ckad-control-plane` on a single-node kind cluster).

The `curl` to the node IP runs from your own machine. That works with `kind` because the node is a
container on a Docker network your host can route to. On the exam, run it from the terminal you're
given, or from a temporary Pod if the node IPs aren't reachable from there.

## Cleanup

```bash
kubectl delete ns jade
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
