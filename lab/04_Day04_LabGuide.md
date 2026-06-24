# Day 04 — Lab Guide: Networking (Ingress, Egress & Cluster Networking)

**Session:** Ingress, Egress & Networking
**Duration:** ~2.5 hours
**Environment:** Two environments are used in this guide — see the table below.

---

## Overview

In this lab you will work through Kubernetes networking the way a support engineer actually encounters it: starting from the cluster's IP ranges, through Services and DNS, out to Ingress and TLS, and finally controlling traffic with NetworkPolicy and an egress gateway pattern. Each lab builds on the previous one's resources — work through them in order.

### Which Cluster Do I Use for Each Lab?

You have two clusters available: **Docker Desktop Kubernetes** (local, you are cluster-admin) and the **shared training cluster** (you're one of many trainees, each mapped to your own namespace, with cluster-wide read access and edit/delete rights only inside your own namespace). Use whichever this table says — mixing them up is the most common source of "it doesn't work" in this lab.

| Lab | Cluster | Why |
|---|---|---|
| Lab 1 — IP Ranges | Shared cluster | You're reading real shared-cluster ranges; Node read access has been granted to your account |
| Lab 2 — Service Types | **Docker Desktop** | `LoadBalancer` behaves correctly out of the box only here; no cloud/MetalLB controller on the shared cluster |
| Lab 3 — DNS & CoreDNS | Shared cluster | CoreDNS behavior is the same everywhere; using the shared cluster keeps you in the habit of reading cluster-wide `kube-system` |
| Lab 4 — Ingress with HAProxy | **Docker Desktop** | Installing an Ingress controller needs cluster-admin (cluster-scoped RBAC objects) — you don't have that on the shared cluster, and 41 trainees installing into the same namespace would collide anyway |
| Lab 5 — TLS on Ingress | **Docker Desktop** | Continues directly from Lab 4's Ingress controller |
| Lab 6 — NetworkPolicy | Shared cluster | NetworkPolicy is namespace-scoped — fully usable with your edit rights in your own namespace |
| Lab 7 — Egress Control | Shared cluster | Same as Lab 6; egress NetworkPolicy is namespace-scoped |

### What You Will Learn

- How to find and read a cluster's node/pod/service IP ranges
- When to use each Service type, hands-on
- How CoreDNS resolves names to ClusterIPs, step by step
- How to configure Ingress routing rules and TLS termination
- How to install and use an Ingress controller (this guide uses **HAProxy Ingress**)
- How to write and verify NetworkPolicy allow/deny rules
- How to reason about and simulate an Egress Gateway pattern

### Before You Begin

- You have completed Day 01–03 labs
- `kubectl` is connected to the cluster, in your assigned namespace
- `helm` is installed (used to install the Ingress controller)
- `dig` or `nslookup` is available, or you're comfortable running them from a debug Pod
- For TLS steps: `openssl` is installed locally

Set a namespace variable you'll reuse throughout:

```bash
export NS=$(kubectl config view --minify -o jsonpath='{..namespace}')
echo "Working in namespace: $NS"
```

---

## Lab 1: Explore the Cluster's IP Ranges

**Cluster:** Shared training cluster

### Summary
Before touching Services or Ingress, find the three address spaces this cluster actually uses: node, pod, and service.

### Steps

**1. Find node IPs (the node network).**

```bash
kubectl get nodes -o wide
```

**2. Find each node's Pod CIDR (the pod network).**

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  ->  "}{.spec.podCIDR}{"\n"}{end}'
```

> Both commands above read `Node` objects, which are cluster-scoped (not namespaced). Your account has been granted a dedicated read-only `ClusterRole`/`ClusterRoleBinding` for `nodes` specifically — the general `view` access you have does not cover them.

**3. Find the cluster's overall pod and service network ranges.**

These come from `kube-apiserver`/`kube-controller-manager` flags (`--service-cluster-ip-range`, `--cluster-cidr`), which only cluster-admin can read directly. **Ask your instructor for these two ranges** rather than trying to read them yourself — your account intentionally doesn't have rights to API server process flags or `kubectl cluster-info dump` (it's also far too heavy a command to run with 40+ trainees on the same cluster at once).

**4. Confirm the service range yourself, indirectly.**

You don't need admin rights to see which range a `ClusterIP` actually comes from — create a throwaway Service in your own namespace and check it:

```bash
kubectl create service clusterip range-check --tcp=80:80
kubectl get svc range-check -o jsonpath='{.spec.clusterIP}{"\n"}'
kubectl delete svc range-check
```

### What to Observe

- Are the node, pod, and service ranges all different from each other?
- Does every node have a distinct Pod CIDR slice?
- Could you tell, just from an IP, which of the three networks it belongs to?
- Why can you read `Node` objects but not the raw API server flags — what's the RBAC distinction?

---

## Lab 2: Service Types in Practice

**Cluster:** Docker Desktop Kubernetes (switch your `kubectl` context — `kubectl config use-context docker-desktop`)

### Summary
Deploy one backend and expose it through each Service type, observing how reachability changes. This lab runs on Docker Desktop, not the shared cluster — `LoadBalancer` only behaves correctly out of the box here, and you have cluster-admin to freely create/delete whatever you need.

### Steps

**1. Deploy a simple backend.**

```bash
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl rollout status deployment/web
```

**2. Expose it as ClusterIP (the default).**

```bash
kubectl expose deployment web --name=web-clusterip --port=80
kubectl get svc web-clusterip
```

Test from inside the cluster only:

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- web-clusterip
```

**3. Expose it as NodePort.**

```bash
kubectl expose deployment web --name=web-nodeport --port=80 --type=NodePort
kubectl get svc web-nodeport
```

On Docker Desktop, the node *is* your machine — hit it directly at `localhost:<assigned-port>` (the port `kubectl get svc web-nodeport` shows under `PORT(S)`, e.g. `80:31234/TCP` → use `31234`).

```bash
curl http://localhost:<assigned-port>/
```

**4. Expose it as LoadBalancer.**

```bash
kubectl expose deployment web --name=web-lb --port=80 --type=LoadBalancer
kubectl get svc web-lb -w
```

Docker Desktop auto-resolves `EXTERNAL-IP` to `localhost` within a few seconds — no tunnel needed. Once it does:

```bash
curl http://localhost/
```

> On the shared training cluster this would behave differently: without a cloud load-balancer controller or MetalLB installed, `EXTERNAL-IP` would stay `<pending>` forever, and you'd fall back to NodePort. Worth remembering for real incidents — "LoadBalancer stuck pending" almost always means no LB controller is wired up for that cluster.

**5. Create a Headless Service and compare DNS.**

```bash
kubectl expose deployment web --name=web-headless --port=80 --cluster-ip=None
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-headless
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-clusterip
```

### What to Observe

- Which Service types were reachable from your laptop, and which only from inside the cluster?
- What did `nslookup` return for the headless Service vs. the ClusterIP Service?
- Match each Service type back to the "when to use" table from the slides — does the behavior match what was taught?

---

## Lab 3: DNS & CoreDNS, Hands-On

**Cluster:** Shared training cluster (switch back: `kubectl config use-context <your-shared-cluster-context>`)

### Summary
Trace a real DNS lookup from a Pod all the way through CoreDNS.

### Steps

**0. Recreate `web` here.** Lab 2's `web` Deployment and `web-clusterip` Service live on Docker Desktop, not here — this lab (and Labs 6/7) need their own copy on the shared cluster, in your namespace:

```bash
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl rollout status deployment/web
kubectl expose deployment web --name=web-clusterip --port=80
```

**1. Find CoreDNS itself.**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc kube-dns -n kube-system
```

**2. Look at a Pod's injected DNS config.**

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- cat /etc/resolv.conf
```

**3. Resolve a Service by short name, and by full name.**

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-clusterip
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-clusterip.$NS.svc.cluster.local
```

**4. Watch CoreDNS logs while you resolve something.**

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns -f
# in another terminal, trigger a lookup, then Ctrl+C the logs
```

**5. Break it on purpose: look up a Service that doesn't exist.**

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup does-not-exist
```

### What to Observe

- What `nameserver` IP is in `/etc/resolv.conf`? Does it match the `kube-dns` Service's ClusterIP?
- What entries are in the `search` line, and why does that make the short name work?
- What does a failed lookup look like — what would a real "DNS fails for a Service" ticket show you?

---

## Lab 4: Ingress Setup with HAProxy Ingress

**Cluster:** Docker Desktop Kubernetes (stay on `docker-desktop` context from Lab 2)

### Summary
Install an Ingress controller and route traffic to your `web` Deployment by host and path. This lab uses **HAProxy Ingress** specifically.

> This lab must run on Docker Desktop, not the shared cluster. Installing an Ingress controller creates cluster-scoped objects (a new `Namespace`, `ClusterRole`/`ClusterRoleBinding` for the controller) — your shared-cluster account only has edit/delete rights inside your own namespace, so this install would fail there with `Forbidden`. Even with admin rights, 40+ trainees installing into the same `ingress-haproxy` namespace name on one shared cluster would collide. On Docker Desktop you're cluster-admin on your own cluster, so this is the right place to practice the full install.

### Steps

**1. Install HAProxy Ingress via Helm.**

```bash
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm repo update
helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
  --create-namespace --namespace ingress-haproxy \
  --set controller.hostNetwork=true
```

**2. Confirm the controller is running.**

```bash
kubectl get pods -n ingress-haproxy
kubectl get svc -n ingress-haproxy
```

**3. Create an Ingress resource routing to your `web` Service.**

```yaml
# web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: haproxy
spec:
  rules:
    - host: web.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-clusterip
                port:
                  number: 80
```

```bash
kubectl apply -f web-ingress.yaml
kubectl get ingress web-ingress
```

**4. Point your local resolver at the controller and test.**

```bash
# find the controller's address (node IP, or LB IP if you ran a tunnel)
kubectl get pods -n ingress-haproxy -o wide
echo "<controller-ip>  web.lab.local" | sudo tee -a /etc/hosts
curl http://web.lab.local/
```

**5. Add a second path rule and confirm both routes work.**

Edit `web-ingress.yaml` to add a `/api` path pointing at `web-nodeport` (or another Service), re-apply, and curl both paths.

### What to Observe

- What happens if you `curl` the host before the `/etc/hosts` entry is added?
- What does `kubectl describe ingress web-ingress` show for backend health?
- HAProxy Ingress uses different annotations than NGINX Ingress — check `haproxy-ingress.github.io` docs for one annotation you might use in production (e.g. rate limiting) and note it here.

---

## Lab 5: TLS Termination on Ingress

**Cluster:** Docker Desktop Kubernetes (continues directly from Lab 4)

### Summary
Issue a self-signed certificate, store it as a Secret, and terminate TLS at the Ingress.

### Steps

**1. Generate a self-signed certificate.**

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout web-tls.key -out web-tls.crt \
  -subj "/CN=web.lab.local/O=lab"
```

**2. Create the TLS Secret.**

```bash
kubectl create secret tls web-tls --cert=web-tls.crt --key=web-tls.key
```

**3. Add a `tls` block to the Ingress.**

```yaml
spec:
  tls:
    - hosts:
        - web.lab.local
      secretName: web-tls
  rules:
    - host: web.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-clusterip
                port:
                  number: 80
```

```bash
kubectl apply -f web-ingress.yaml
```

**4. Test over HTTPS.**

```bash
curl -k https://web.lab.local/
```

**5. Inspect the certificate the server actually presents.**

```bash
openssl s_client -connect web.lab.local:443 -servername web.lab.local </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

### What to Observe

- Why is `-k` needed on the `curl` command? What would a real deployment use instead (hint: `cert-manager`)?
- Does the certificate's `CN`/`subject` match the host you requested?
- What happens if you `curl https://` a *different* hostname against the same Ingress?

---

## Lab 6: NetworkPolicy — Default-Deny and Allow

**Cluster:** Shared training cluster

### Summary
Lock down a namespace by default, then open a specific, intentional hole.

### Steps

**1. Deploy a second app to represent "another team."**

```bash
kubectl create deployment other-team-app --image=nginx:alpine
kubectl expose deployment other-team-app --port=80
```

**2. Confirm `web` is reachable from anywhere right now (the default).**

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 web-clusterip
```

**3. Apply a default-deny-all-ingress policy.**

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
```

```bash
kubectl apply -f default-deny.yaml
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 web-clusterip
```

**4. Allow traffic to `web` only from Pods labeled `role: frontend`.**

```yaml
# allow-frontend-to-web.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-web
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
```

```bash
kubectl apply -f allow-frontend-to-web.yaml
```

**5. Verify: unlabeled traffic is still blocked, labeled traffic gets through.**

```bash
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 web-clusterip
kubectl run debug --labels="role=frontend" --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 web-clusterip
```

### What to Observe

- After step 3, did the request from step 2 still succeed?
- After step 4, which of the two `wget` attempts in step 5 succeeded?
- Did `other-team-app` need any policy at all? Why or why not?

---

## Lab 7: Egress Traffic Control

**Cluster:** Shared training cluster

### Summary
Restrict outbound traffic with an egress `NetworkPolicy`, then simulate the egress gateway pattern from the slides — funneling traffic through one node/pod so it leaves with a single, predictable identity.

> Note: a production Egress Gateway (Cilium/Calico Enterprise/Istio) needs CNI-level support this training cluster may not have. Steps 1–3 use real `NetworkPolicy` egress rules. Steps 4–5 simulate the *pattern* with a forward proxy, so you can observe the effect even without that CNI feature.

### Steps

**1. Confirm `web` can reach the internet right now.**

```bash
kubectl exec deploy/web -- wget -qO- --timeout=3 https://example.com >/dev/null && echo "reached internet"
```

**2. Apply an egress policy that blocks all outbound traffic except DNS.**

```yaml
# deny-egress-except-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-except-dns
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f deny-egress-except-dns.yaml
kubectl exec deploy/web -- wget -qO- --timeout=3 https://example.com
```

**3. Allow `web` to reach one specific external-style target, nothing else.**

Add an extra `to` block (e.g. a fixed IP or another in-cluster Service standing in for the "partner API") and re-test.

**4. Simulate an egress gateway: stand up a single forward-proxy Pod.**

```bash
kubectl run egress-gw --image=nginx:alpine --labels="role=egress-gateway"
kubectl expose pod egress-gw --port=8080 --target-port=80
```

This Pod represents "the one place outbound traffic leaves through." In a real Egress Gateway, your CNI would transparently route tagged Pods' egress traffic here and SNAT it to one fixed IP.

**5. Discuss/observe the effect.**

```bash
kubectl get pod egress-gw -o wide   # note its single Node/IP
```

Talk through: if every outbound call from `web` were forced through `egress-gw`, what single source IP would a partner API see, regardless of which underlying `web` replica or node made the call?

### What to Observe

- After step 2, did the request to `example.com` succeed or fail? Was DNS still working?
- Why does an egress policy alone *not* give you a fixed source IP — what does the gateway add on top?
- What would you check on a ticket like "partner API suddenly rejecting our calls" after today?

---

## Cleanup

Run each block against the cluster it was created on.

**On Docker Desktop (`kubectl config use-context docker-desktop`) — Labs 2, 4, 5:**

```bash
kubectl delete ingress web-ingress --ignore-not-found
kubectl delete secret web-tls --ignore-not-found
kubectl delete svc web-clusterip web-nodeport web-lb web-headless --ignore-not-found
kubectl delete deployment web --ignore-not-found
helm uninstall haproxy-ingress -n ingress-haproxy
kubectl delete namespace ingress-haproxy --ignore-not-found
rm -f web-ingress.yaml web-tls.key web-tls.crt
sudo sed -i '/web.lab.local/d' /etc/hosts
```

**On the shared training cluster (your own namespace) — Labs 1, 3, 6, 7:**

```bash
kubectl delete networkpolicy default-deny-ingress allow-frontend-to-web deny-egress-except-dns --ignore-not-found
kubectl delete svc web-clusterip egress-gw --ignore-not-found
kubectl delete deployment web other-team-app --ignore-not-found
kubectl delete pod egress-gw --ignore-not-found
rm -f default-deny.yaml allow-frontend-to-web.yaml deny-egress-except-dns.yaml
```

> `web` / `web-clusterip` here are the copies created in Lab 3, Step 0 — separate from the Lab 2 copies on Docker Desktop, which are cleaned up in the block above.

---

## Optional Stretch Tasks

### Stretch 1: Compare Ingress Controllers

- Install NGINX Ingress alongside HAProxy Ingress in a separate namespace.
- Route a second host through it.
- Compare the annotations each one needs for the same path-rewrite behavior.

### Stretch 2: IPVS vs iptables

- Check which kube-proxy mode your cluster runs (`kubectl get cm kube-proxy -n kube-system -o yaml | grep mode`).
- If possible, inspect the actual rules (`iptables -t nat -L` or `ipvsadm -Ln` on a node) for one of your Services.

### Stretch 3: Confirm Your CNI's Egress Gateway Support (or Lack of It)

This training cluster was provisioned with Kubespray, which defaults to **Calico**. Check it yourself:

```bash
kubectl get pods -n kube-system | grep -i calico
```

Calico **open-source** (Kubespray's default) does **not** support Egress Gateway — that's a Calico Enterprise/Calico Cloud-only feature. So on this cluster, Lab 7 Steps 4–5's forward-proxy Pod is a deliberate simulation, not a shortcut around a real feature you're missing.

- `CustomResourceDefinition` is cluster-scoped, like `Node` — your account isn't granted CRD read, so don't expect `kubectl get crd` to work here. Ask your instructor to confirm there's no `*EgressGatewayPolicy`-style CRD installed, or take their word that open-source Calico doesn't ship one.
- Write down, in your own words, why an `Egress Gateway` is a CNI-level (or CNI-vendor-tier) feature rather than something `NetworkPolicy` alone can express — tie it back to the "every node exits with a different IP" point from the slides.
- If you have access to a Cilium or Calico Enterprise cluster elsewhere (e.g. a personal Docker Desktop install with Cilium as the CNI), try a real `CiliumEgressGatewayPolicy` there and compare the setup to the simulation here.

### Stretch 4: Headless Service + StatefulSet

- Deploy a 3-replica StatefulSet behind a headless Service.
- Resolve each Pod's individual DNS name and confirm they're stable across a Pod restart.

---

## Self-Check Questions

1. Which of the three cluster IP ranges (node/pod/service) would a `ClusterIP` Service's address come from?
2. Why does `nslookup` return multiple IPs for a headless Service but one for a ClusterIP Service?
3. What's the difference between an `Ingress` object having no matching Pods and an Ingress controller not being installed at all — how would each look from `kubectl describe ingress`?
4. Why does a default-deny `NetworkPolicy` not need an explicit `podSelector` value to apply to every Pod in the namespace?
5. An egress `NetworkPolicy` allows a Pod to call an external API, but the partner says they still see different source IPs on different requests. What's missing?
