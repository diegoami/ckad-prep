# 074 — Fix two drafted NetworkPolicies: a `from` list that ORs, and an egress `ipBlock` without DNS

**Domain:** Services and Networking · **Difficulty:** Hard

`050`, `057` and `070` only use `podSelector` peers. This one mixes in a `namespaceSelector`, a
named port and an egress `ipBlock` with `except`. Both drafted policies apply without error and
both are wrong: one lets far too much in, because two peers that were meant to be one condition
were written as separate list entries, and the other locks its Pods out of DNS.

## Task

> The Draco team left two draft NetworkPolicies in `~/ckad/074/` for namespace `draco`:
>
> - `metrics-store-policy.yaml` (policy `metrics-store-ingress`): the `metrics-store` Pods (label
>   `app=metrics-store`) may accept connections on their port named `metrics` **only** from Pods
>   labelled `role=scraper` that run in the monitoring namespace `hydra`.
> - `webhook-relay-policy.yaml` (policy `webhook-relay-egress`): the `webhook-relay` Pods (label
>   `app=webhook-relay`) may open outbound connections only to the partner network
>   `172.16.0.0/12` on TCP 443, never to the quarantined subnet `172.20.5.0/24` inside it, and
>   must still be able to resolve DNS names.
>
> Apply both, test that each one does what it's meant to, and fix any policy that doesn't. Keep the
> policy names, and leave the fixed versions in the same files.

## Documentation

What to look up: **Network Policies**, specifically the AND/OR semantics of the `from` list and
`ipBlock`.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors> —
  one `from` element holding both `namespaceSelector` and `podSelector` is an AND; two elements are
  an OR.
- <https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-egress-traffic> —
  a Pod selected by a policy with `Egress` in `policyTypes` may only make the connections that
  policy (or another one selecting it) lists.

## Setup

```bash
kubectl create ns draco
kubectl create ns hydra
kubectl label ns hydra purpose=monitoring

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: metrics-store
  namespace: draco
  labels:
    app: metrics-store
spec:
  containers:
  - name: store
    image: busybox:1.36
    command: ["nc", "-lk", "-p", "9090", "-e", "true"]
    ports:
    - name: metrics
      containerPort: 9090
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-store
  namespace: draco
spec:
  selector:
    app: metrics-store
  ports:
  - port: 9090
    targetPort: metrics
EOF
kubectl run cost-exporter --image=busybox:1.36 -n draco --labels=role=scraper --command -- sleep 86400
kubectl run webhook-relay --image=busybox:1.36 -n draco --labels=app=webhook-relay --command -- sleep 86400
kubectl run scraper --image=busybox:1.36 -n hydra --labels=role=scraper --command -- sleep 86400
kubectl run dashboard --image=busybox:1.36 -n hydra --labels=role=dashboard --command -- sleep 86400
kubectl wait --for=condition=Ready pod --all -n draco --timeout=60s
kubectl wait --for=condition=Ready pod --all -n hydra --timeout=60s

mkdir -p ~/ckad/074
cat > ~/ckad/074/metrics-store-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metrics-store-ingress
  namespace: draco
spec:
  podSelector:
    matchLabels:
      app: metrics-store
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    - podSelector:
        matchLabels:
          role: scraper
    ports:
    - protocol: TCP
      port: metrics
EOF

cat > ~/ckad/074/webhook-relay-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: webhook-relay-egress
  namespace: draco
spec:
  podSelector:
    matchLabels:
      app: webhook-relay
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 172.16.0.0/12
        except:
        - 172.20.5.0/24
    ports:
    - protocol: TCP
      port: 443
EOF
```

The blocking described below needs a CNI that enforces NetworkPolicy. kindnet in kind v0.23 and
earlier doesn't, so every connection succeeds there whatever the policies say: you can practise the
steps but not observe the effect. Recent kind releases enforce it. See `guide/practice-cluster.md`.
The `#` comments below show what an enforcing CNI is expected to do.

## Solution — the metrics-store policy

Apply it as drafted, then test from three clients: the one that should get in, and two that
shouldn't.
```bash
kubectl apply -f ~/ckad/074/metrics-store-policy.yaml

kubectl exec -n hydra scraper -- nc -zv -w 3 metrics-store.draco 9090
# metrics-store.draco (10.96.121.73:9090) open   <- correct: role=scraper in hydra
kubectl exec -n hydra dashboard -- nc -zv -w 3 metrics-store.draco 9090
# open as well   <- WRONG: not a scraper
kubectl exec -n draco cost-exporter -- nc -zv -w 3 metrics-store 9090
# open as well   <- WRONG: a scraper, but not in hydra
```

**The trap:** under `ingress[].from`, `namespaceSelector` and `podSelector` were written as **two
separate list entries**, each with its own leading `-`. Separate entries in a `from` list are OR'd:
a client only has to match *one* of them. So the draft admits:
- every Pod in any namespace labelled `purpose=monitoring`, whatever its role (`dashboard`);
- every `role=scraper` Pod in the policy's **own** namespace, because a `podSelector` entry with no
  `namespaceSelector` next to it means "in the same namespace as this NetworkPolicy"
  (`cost-exporter`).

The intent is a single condition, "role=scraper **and** in hydra", and that needs both selectors in
the **same** entry. The fix is a one-character edit: replace the `-` in front of `podSelector` with a
space, so it lines up with `namespaceSelector` as a second key of the same entry:
```bash
cat > ~/ckad/074/metrics-store-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metrics-store-ingress
  namespace: draco
spec:
  podSelector:
    matchLabels:
      app: metrics-store
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
      podSelector:
        matchLabels:
          role: scraper
    ports:
    - protocol: TCP
      port: metrics
EOF
kubectl apply -f ~/ckad/074/metrics-store-policy.yaml

kubectl exec -n hydra scraper -- nc -zv -w 3 metrics-store.draco 9090
# open — still correct
kubectl exec -n hydra dashboard -- nc -zv -w 3 metrics-store.draco 9090
# no "open" line; exits non-zero after the 3s timeout
kubectl exec -n draco cost-exporter -- nc -zv -w 3 metrics-store 9090
# no "open" line; exits non-zero after the 3s timeout
```
Check the structure with `kubectl describe`, which spells the difference out:
```bash
kubectl describe networkpolicy metrics-store-ingress -n draco | grep -A4 'From:'
#     From:
#       NamespaceSelector: purpose=monitoring
#       PodSelector: role=scraper
```
Both selectors sit under one `From:`. Describe the draft version and you get two `From:` blocks
instead, one with only the `NamespaceSelector` and one with only the `PodSelector`: a quick way
to spot the OR without counting dashes.

`port: metrics` is a **named port**: it matches whatever container port is called `metrics` on the
selected Pods (9090 here), so the policy keeps working if the port number changes. It refers to the
Pod's port, not the Service's; the Service here also targets the port by name.

## Solution — the webhook-relay egress policy

```bash
kubectl apply -f ~/ckad/074/webhook-relay-policy.yaml
kubectl exec -n draco webhook-relay -- nslookup -timeout=3 kubernetes.default.svc.cluster.local
# expected on an enforcing CNI: no answer, the lookup times out
```
**The trap:** once an egress policy selects a Pod, every outbound connection not listed is denied,
and that includes DNS. The cluster DNS Pods (CoreDNS behind `kube-dns` in `kube-system`) live in
the Pod network (10.244.0.0/16 on kind), outside `172.16.0.0/12`, and on port 53, not 443. So the
relay can no longer resolve a single name. (Use the full name with busybox's `nslookup`: it
ignores the Pod's DNS search domains, so a short name like `kubernetes.default` returns `NXDOMAIN`
even when DNS works.) Add a second egress rule for port 53 on both protocols:
```bash
cat > ~/ckad/074/webhook-relay-policy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: webhook-relay-egress
  namespace: draco
spec:
  podSelector:
    matchLabels:
      app: webhook-relay
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 172.16.0.0/12
        except:
        - 172.20.5.0/24
    ports:
    - protocol: TCP
      port: 443
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
kubectl apply -f ~/ckad/074/webhook-relay-policy.yaml
kubectl exec -n draco webhook-relay -- nslookup -timeout=3 kubernetes.default.svc.cluster.local
# Name:	kubernetes.default.svc.cluster.local
# Address: 10.96.0.1
```
The DNS rule has no `to`, so it allows port 53 to any destination. To narrow it to the cluster DNS,
add `to: [{namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}, podSelector:
{matchLabels: {k8s-app: kube-dns}}}]`, one entry with both selectors, the same AND as above.

The `ipBlock` part was right as drafted. `except` carves a hole out of the allowed CIDR:
`172.16.0.0/12` allows 172.16.0.0–172.31.255.255, and `172.20.5.0/24` blocks those 256 addresses
and nothing more. The API server checks that the exception really sits inside the CIDR; an
exception outside it is rejected with `must be a strict subset of cidr`. There's no practical way
to put a real destination in `172.20.5.0/24` on a kind cluster, so you can't watch this part fire;
the rule is for keeping a known-bad range out of an otherwise allowed network.

## Cleanup

```bash
kubectl delete ns draco hydra
rm -rf ~/ckad/074
```

*Verified on kind v0.23 (Kubernetes v1.30) on 2026-09-24: Setup, Solution and Cleanup run as
written and both policies apply cleanly, but enforcement wasn't observable because kindnet on kind
v0.23 doesn't enforce NetworkPolicy.*
