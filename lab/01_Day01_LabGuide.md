# Day 01 — Lab Guide: Kubernetes Architecture & Setup

**Session:** Kubernetes Architecture & Setup  
**Duration:** ~2.5 hours  
**Environment:** Pre-provisioned shared cluster  

---

## Overview

In this lab, you will explore a live Kubernetes cluster and get hands-on with the `kubectl` command-line tool. By the end, you'll be comfortable navigating a cluster, understanding what's running inside it, and using the key commands that support engineers use every day.

### What You Will Learn

- How to connect to and inspect a Kubernetes cluster
- How namespaces and contexts work
- How to discover Kubernetes resources and read their schemas
- How to identify control plane components
- How to inspect pods, read logs, and trace events
- How to verify RBAC permissions for support roles

### Before You Begin

- You have been provided with cluster access credentials
- `kubectl` is installed and configured on your machine
- Run `kubectl cluster-info` to confirm you are connected

---

## Lab 1: Connect to the Cluster

In this lab, you'll verify connectivity and understand the cluster topology.

### Steps

**1. Verify your cluster connection.**

```bash
kubectl cluster-info
```

You should see the API server and CoreDNS endpoint URLs. This confirms your `kubectl` is pointed at the right cluster.

**2. List all nodes in the cluster.**

```bash
kubectl get nodes
```

You should see one or more nodes with a `Ready` status. Note the `ROLES` column — nodes may be labelled `control-plane` or `worker`.

**3. Get extended node details.**

```bash
kubectl get nodes -o wide
```

This adds columns like `INTERNAL-IP`, `OS-IMAGE`, `KERNEL-VERSION`, and `CONTAINER-RUNTIME`. Take a moment to read the runtime version — it tells you whether the cluster uses `containerd` or `docker`.

**4. Check individual node health.**

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"	"}{.status.conditions[?(@.type=="Ready")].status}{"
"}{end}'
```

Each line shows a node name and whether its `Ready` condition is `True` or `False`. This is a quick health check without needing a full describe.

### What to Observe

- How many nodes are in this cluster?
- Which node is the control plane?
- What container runtime is being used?

---

## Lab 2: Namespaces and Contexts

Namespaces let you logically separate resources inside a cluster. Contexts let you switch between different clusters or configurations.

### Steps

**1. List all namespaces.**

```bash
kubectl get namespaces
```

You'll see at minimum: `default`, `kube-system`, `kube-public`, and `kube-node-lease`. These are created by Kubernetes automatically.

**2. View all pods across all namespaces.**

```bash
kubectl get pods -A
```

The `-A` flag means "all namespaces". Notice the first column is `NAMESPACE` — this gives you a full picture of everything running in the cluster.

**3. Switch to the `kube-system` namespace.**

```bash
kubectl config set-context --current --namespace=kube-system
```

**4. Verify you are now in `kube-system`.**

```bash
kubectl config view --minify | grep namespace
```

You should see `namespace: kube-system`.

**5. Switch back to `default`.**

```bash
kubectl config set-context --current --namespace=default
```

**6. View all available contexts.**

```bash
kubectl config get-contexts
```

The `*` marks your active context. If you are managing multiple clusters, you would switch between them using:

```bash
kubectl config use-context <cluster-name>
```

### What to Observe

- What namespaces exist in the cluster?
- Can you see pods in `kube-system` when you run `kubectl get pods -A`?

---

## Lab 3: Exploring API Resources

Every object in Kubernetes — pods, services, deployments — is an API resource. This lab helps you understand what's available and how to read resource definitions.

### Steps

**1. List all available API resources.**

```bash
kubectl api-resources
```

This lists every resource type the cluster supports. You'll see names, shortnames, API groups, and whether each is namespaced.

**2. View shortnames only.**

```bash
kubectl api-resources --short-names
```

Shortnames like `po` for pods and `svc` for services save typing. You'll use these often in daily support work.

**3. Explore the Pod resource schema.**

```bash
kubectl explain pod
```

This shows the top-level fields of a Pod object. Think of it as live documentation — no internet needed.

**4. Go deeper into the Pod spec.**

```bash
kubectl explain pod.spec
```

**5. Explore container fields.**

```bash
kubectl explain pod.spec.containers
```

This is useful when you need to understand what fields are required or optional for a container definition.

**6. Identify namespaced vs cluster-scoped resources.**

```bash
# Resources inside namespaces
kubectl api-resources --namespaced=true

# Resources that exist cluster-wide
kubectl api-resources --namespaced=false
```

Cluster-scoped resources like `nodes` and `persistentvolumes` are not tied to any namespace.

### What to Observe

- What shortname is used for `deployments`?
- Is `namespace` itself a namespaced resource?
- What fields are required in `pod.spec.containers`?

---

## Lab 4: Inspecting the kube-system Namespace

`kube-system` is where Kubernetes runs its own internal components. As a support engineer, you'll visit this namespace regularly.

### Steps

**1. List all pods in `kube-system`.**

```bash
kubectl get pods -n kube-system
```

You'll see the core control plane components running as pods. Note their names — they typically include the node name as a suffix.

**2. Get extended pod details.**

```bash
kubectl get pods -n kube-system -o wide
```

This shows which node each pod is running on. Control plane pods should all be on the control-plane node.

**3. Describe a CoreDNS pod.**

```bash
kubectl describe pod coredns -n kube-system
```

Look at the `Events` section at the bottom — this tells you the lifecycle history of the pod. Also note `Containers`, `Volumes`, and `Conditions`.

**4. View recent events in `kube-system`.**

```bash
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

Events give you a timeline of what the cluster has been doing. Useful for spotting restarts, scheduling failures, or image pull issues.

**5. View API server logs.**

```bash
kubectl logs -n kube-system kube-apiserver-control-plane
```

> These logs can be verbose. Scroll through a few lines and press `Ctrl+C` to exit.

**6. List pods with their component labels.**

```bash
kubectl get pods -n kube-system -o custom-columns=NAME:.metadata.name,ROLE:.metadata.labels."k8s\.app"
```

### Component Reference

| Pod Name | Purpose |
|---|---|
| `coredns` | DNS service discovery for the cluster |
| `kube-apiserver` | Main entry point — all kubectl commands hit this |
| `kube-controller-manager` | Keeps desired state in sync |
| `kube-scheduler` | Decides which node a pod runs on |
| `etcd` | Key-value store for all cluster data |

### What to Observe

- How many times has each pod restarted?
- Which node is the API server running on?
- What events appear in `kube-system`?

---

## Lab 5: Resource Discovery and Inspection

In this lab, you'll learn how to filter resources, read their full specs, and understand output formats.

### Steps

**1. List pods in the default namespace.**

```bash
kubectl get pods
```

**2. Show labels attached to pods.**

```bash
kubectl get pods --show-labels
```

Labels are key-value pairs attached to resources. They are used for grouping, filtering, and targeting by services.

**3. Filter pods by label.**

```bash
kubectl get pods -l app=myapp
```

This returns only pods that have the label `app=myapp`. Change the value to match what you see in your cluster.

**4. Describe a pod.**

```bash
kubectl describe pod <pod-name>
```

Replace `<pod-name>` with a real pod name from the list. The describe output shows IP, node, containers, volumes, events, and more.

**5. View the full YAML of a pod.**

```bash
kubectl get pod <pod-name> -o yaml
```

This is the live manifest — what Kubernetes is actually running. You can use this to understand how a resource was configured.

**6. List all resources in a namespace.**

```bash
kubectl get all -n default
```

`get all` returns pods, services, deployments, and replica sets in one command.

**7. Count pods across namespaces.**

```bash
kubectl get pods -A -o name | cut -d/ -f1 | sort | uniq -c
```

### What to Observe

- What labels are on the pods in your namespace?
- What is the difference between `describe` and `get -o yaml`?

---

## Lab 6: Logs, Events, and Exec

These are the three commands you'll use most often when something goes wrong. Get comfortable with them now.

### Steps

**1. Check pod status.**

```bash
kubectl get pods
```

Look at the `STATUS` column. Common states: `Running`, `Pending`, `CrashLoopBackOff`, `OOMKilled`.

**2. Describe a pod and read its events.**

```bash
kubectl describe pod <pod-name>
```

Scroll to the bottom. The `Events` section is the most useful part — it shows the lifecycle of the pod in chronological order.

**3. Read pod logs.**

```bash
kubectl logs <pod-name>
```

**4. Read logs from a specific container.**

```bash
kubectl logs <pod-name> -c <container-name>
```

Use this when a pod has multiple containers (e.g., a main app + a sidecar).

**5. Filter events for a specific pod.**

```bash
kubectl get events --field-selector involvedObject.name=<pod-name>
```

**6. Exec into a running pod.**

```bash
kubectl exec -it <pod-name> -- ls /
```

This opens an interactive shell inside the container. Try running `env`, `cat /etc/hosts`, or `curl localhost` if the application is running.

**7. Check resource usage.**

```bash
kubectl top pods
kubectl top nodes
```

> Note: This requires `metrics-server` to be installed on the cluster.

### What to Observe

- What do the pod logs say?
- Can you identify what stage a pod failed at using Events?

---

## Lab 7: Container Runtime Verification

Kubernetes delegates container execution to a container runtime. This lab shows you how to identify and inspect the runtime.

### Steps

**1. Check the container runtime via kubectl.**

```bash
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
```

You should see something like `containerd://1.6.24`.

**2. On the node, list running containers using crictl.**

```bash
sudo crictl ps
```

**3. Check the runtime version.**

```bash
sudo crictl version
```

**4. Inspect a specific container.**

```bash
sudo crictl inspect <container-id>
```

**5. View container logs via crictl.**

```bash
sudo crictl logs <container-id>
```

### What to Observe

- What runtime is the cluster using?
- How does the crictl container list map to kubectl pod list?

---

## Lab 8: RBAC — Understanding Your Access

RBAC (Role-Based Access Control) governs what actions users and service accounts can perform in Kubernetes. As a support engineer, you need to understand your permissions and how to check them.

### Steps

**1. List roles in the default namespace.**

```bash
kubectl get roles -n default
```

**2. List cluster-wide roles.**

```bash
kubectl get clusterroles
```

**3. List role bindings.**

```bash
kubectl get rolebindings -n default
```

**4. Describe a role to see what it allows.**

```bash
kubectl describe role support-reader -n default
```

**5. Check if a user can perform an action.**

```bash
kubectl auth can-i get pods --as support-user
kubectl auth can-i delete pods --as support-user
kubectl auth can-i list pods --namespace=default --as support-user
```

**6. Check your own current permissions.**

```bash
kubectl auth can-i list pods
kubectl auth can-i delete pods
```

**7. Verify your current user context.**

```bash
kubectl config view --minify | grep user
```

### What to Observe

- What verbs does the `support-reader` role allow?
- Can `support-user` delete pods?
- What's the difference between a `Role` and a `ClusterRole`?

---

## Lab 9: Create a Support RBAC Role

Now you'll create a read-only support role yourself.

### Steps

**1. Create a namespace-scoped Role.**

```bash
kubectl create role support-reader   --verb=get,list,watch   --resource=pods,services,configmaps,secrets,endpoints,events   -n default
```

**2. Bind the role to a user.**

```bash
kubectl create rolebinding support-binding   --role=support-reader   --user=support-user   -n default
```

**3. Verify the role was created.**

```bash
kubectl get role support-reader -n default -o yaml
```

**4. Verify the binding.**

```bash
kubectl get rolebinding support-binding -n default -o yaml
```

**5. Test the permissions.**

```bash
kubectl auth can-i get pods -n default --as support-user
# Expected: yes

kubectl auth can-i delete pods -n default --as support-user
# Expected: no
```

### What to Observe

- Does `support-user` have `get` access to pods?
- Does `support-user` have `delete` access?
- What would you change to give access across all namespaces?

---

## Troubleshooting Reference

| Symptom | Likely Cause | What to Try |
|---|---|---|
| `Unable to connect to server` | Cluster not reachable | Check VPN, `kubectl cluster-info` |
| `No resources found` | Wrong namespace | Add `-n <namespace>` or `-A` |
| `Error from server (NotFound)` | Resource doesn't exist | Check name with `kubectl get pods` |
| `Error from server (Forbidden)` | RBAC restriction | Check role with `kubectl auth can-i` |
| `crictl: command not found` | Not on a node | Use `kubectl exec` approach instead |

---

## Summary

By completing this lab, you have:

- Connected to and inspected a live Kubernetes cluster
- Navigated namespaces and contexts
- Explored the full API resource catalogue
- Identified and understood control plane pods
- Inspected pod logs, events, and resource specs
- Verified container runtimes
- Read and created RBAC roles for support access

---

## Self-Check Questions

Answer these before moving on:

1. What is the difference between a control-plane node and a worker node?
2. What are the four default namespaces in a Kubernetes cluster?
3. Which kubectl command gives you live documentation for any resource field?
4. What does `kubectl describe` show that `kubectl get` does not?
5. What is the difference between a `Role` and a `ClusterRole`?

---

**End of Day 01 Lab Guide**
