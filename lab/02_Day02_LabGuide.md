# Day 02 — Lab Guide: Kubernetes Workloads

**Session:** Kubernetes Workloads  
**Duration:** ~2.5 hours  
**Environment:** Pre-provisioned shared cluster  

---

## Overview

In this lab you will work with the core Kubernetes workload types. You will create and inspect Pods, Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs. You will also inject configuration into pods using ConfigMaps and Secrets.

### What You Will Learn

- How to create and manage different workload types
- How to observe pod lifecycle phases and events
- How to pass configuration into pods using ConfigMaps and Secrets
- How to interpret resource requests, limits, and QoS classes

### Before You Begin

- You have completed Day 01 labs
- Your `kubectl` is connected to the cluster
- You are working in your assigned namespace

---

## Lab 1: Working with Pods

### Summary
Create a simple pod, observe its lifecycle, and inspect it using kubectl.

### Steps

**1. Create a pod manifest.**

```yaml
# pod-basic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: nginx
    image: nginx:1.24
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod-basic.yaml
```

**2. Watch the pod come up.**

```bash
kubectl get pods -w
```

You will see the pod move through `Pending` → `ContainerCreating` → `Running`. Press `Ctrl+C` to stop watching.

**3. Describe the pod.**

```bash
kubectl describe pod my-pod
```

Read the `Events` section carefully. Note the scheduling, image pull, and container start events.

**4. Read the pod logs.**

```bash
kubectl logs my-pod
```

**5. Exec into the pod.**

```bash
kubectl exec -it my-pod -- bash
```

Inside the container, run:
```bash
curl localhost
hostname
env
exit
```

**6. Delete the pod.**

```bash
kubectl delete pod my-pod
```

Observe that it is gone and does not restart. This is the key difference from workloads managed by a controller.

### What to Observe

- What events appear in the `describe` output?
- What happens to the pod after it is deleted — does it come back?

---

## Lab 2: Multi-Container Pod with a Sidecar

### Summary
Create a pod with a main container and a sidecar container sharing a volume.

### Steps

**1. Create the manifest.**

```yaml
# pod-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "while true; do echo $(date) >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: log-sidecar
    image: busybox:1.35
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```

```bash
kubectl apply -f pod-sidecar.yaml
```

**2. Check both containers are running.**

```bash
kubectl get pod sidecar-pod
```

The `READY` column should show `2/2`.

**3. Read logs from the sidecar container.**

```bash
kubectl logs sidecar-pod -c log-sidecar -f
```

You will see log lines written by the app container appearing in the sidecar. Press `Ctrl+C` to stop.

**4. Read logs from the app container.**

```bash
kubectl logs sidecar-pod -c app
```

### What to Observe

- Both containers share the same volume. How would this help in a real logging scenario?
- What does the `READY` column tell you about multi-container pods?

---

## Lab 3: Deployment

### Summary
Create a Deployment, observe the ReplicaSet it creates, and simulate a pod failure.

### Steps

**1. Create a Deployment.**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
```

```bash
kubectl apply -f deployment.yaml
```

**2. Check the Deployment and its pods.**

```bash
kubectl get deployment web-app
kubectl get pods -l app=web-app
```

**3. Inspect the ReplicaSet.**

```bash
kubectl get replicaset
kubectl describe replicaset
```

Note that the ReplicaSet was automatically created by the Deployment.

**4. Simulate a pod failure.**

```bash
# Delete one pod
kubectl delete pod <pod-name>

# Immediately watch what happens
kubectl get pods -l app=web-app -w
```

The Deployment controller detects the missing pod and creates a replacement. This is self-healing in action.

**5. Scale the Deployment.**

```bash
kubectl scale deployment web-app --replicas=5
kubectl get pods -l app=web-app
```

**6. Scale back down.**

```bash
kubectl scale deployment web-app --replicas=3
```

### What to Observe

- How quickly does Kubernetes replace the deleted pod?
- What is the relationship between Deployment, ReplicaSet, and Pod?

---

## Lab 4: StatefulSet

### Summary
Deploy a StatefulSet and observe the stable pod identity that distinguishes it from a Deployment.

### Steps

**1. Create a StatefulSet.**

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  serviceName: "web-stateful"
  replicas: 3
  selector:
    matchLabels:
      app: web-stateful
  template:
    metadata:
      labels:
        app: web-stateful
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
```

```bash
kubectl apply -f statefulset.yaml
```

**2. Observe pod names.**

```bash
kubectl get pods -l app=web-stateful
```

Notice the pod names: `web-stateful-0`, `web-stateful-1`, `web-stateful-2`. These are predictable and stable, unlike Deployment pod names.

**3. Delete a pod and observe recreation.**

```bash
kubectl delete pod web-stateful-1
kubectl get pods -l app=web-stateful -w
```

The pod is recreated with the exact same name `web-stateful-1`.

**4. Check creation order.**

```bash
kubectl describe statefulset web-stateful
```

Note in the Events that pods are created sequentially (0 → 1 → 2).

### What to Observe

- What is the naming pattern for StatefulSet pods?
- How does pod recreation differ from a Deployment?

---

## Lab 5: DaemonSet

### Summary
Inspect a DaemonSet already running in the cluster and understand why it has one pod per node.

### Steps

**1. List existing DaemonSets in kube-system.**

```bash
kubectl get daemonsets -n kube-system
```

**2. Describe a DaemonSet.**

```bash
kubectl describe daemonset kube-proxy -n kube-system
```

Note the `Desired`, `Current`, and `Ready` counts — they match the number of nodes.

**3. Verify one pod per node.**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

Each pod runs on a different node.

**4. Create your own DaemonSet.**

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
      - name: monitor
        image: busybox:1.35
        command: ["sh", "-c", "while true; do echo Node: $(hostname); sleep 10; done"]
```

```bash
kubectl apply -f daemonset.yaml
kubectl get pods -l app=node-monitor -o wide
```

### What to Observe

- How many pods did the DaemonSet create?
- Does the count match the number of nodes?

---

## Lab 6: Job and CronJob

### Summary
Run a one-time Job and then schedule a recurring CronJob.

### Steps

**1. Create a Job.**

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.35
        command: ["sh", "-c", "echo Hello from Job; sleep 5"]
      restartPolicy: OnFailure
```

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl get pods
```

**2. Check the job output.**

```bash
kubectl logs -l job-name=hello-job
```

**3. Verify the job completed.**

```bash
kubectl get job hello-job
```

The `COMPLETIONS` column should show `1/1`.

**4. Create a CronJob.**

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.35
            command: ["sh", "-c", "echo CronJob run at $(date)"]
          restartPolicy: OnFailure
```

```bash
kubectl apply -f cronjob.yaml
```

**5. Watch it run.**

```bash
kubectl get cronjob hello-cron
kubectl get jobs -w
```

Wait 2 minutes and observe a new Job being created automatically.

**6. Check the logs.**

```bash
kubectl logs -l job-name=<latest-job-name>
```

### What to Observe

- What happens to the pod after the Job completes?
- How many Job runs can you see in history?

---

## Lab 7: ConfigMap

### Summary
Create a ConfigMap and inject it into a pod as environment variables and as a mounted file.

### Steps

**1. Create a ConfigMap.**

```bash
kubectl create configmap app-config   --from-literal=APP_ENV=production   --from-literal=LOG_LEVEL=info   --from-literal=MAX_RETRIES=3
```

**2. Inspect the ConfigMap.**

```bash
kubectl get configmap app-config
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

**3. Inject as environment variables.**

```yaml
# pod-configmap-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-config-env
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "env | grep APP && env | grep LOG && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

```bash
kubectl apply -f pod-configmap-env.yaml
kubectl logs pod-config-env
```

**4. Mount as a volume.**

```yaml
# pod-configmap-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-config-vol
spec:
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "ls /etc/config && cat /etc/config/APP_ENV && sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
```

```bash
kubectl apply -f pod-configmap-vol.yaml
kubectl logs pod-config-vol
```

### What to Observe

- How does env var injection differ from volume mount injection?
- What file names appear under `/etc/config`?

---

## Lab 8: Secret

### Summary
Create a Secret, inject it into a pod, and understand the base64 encoding.

### Steps

**1. Create a Secret.**

```bash
kubectl create secret generic db-secret   --from-literal=DB_USER=admin   --from-literal=DB_PASSWORD=supersecret123
```

**2. Inspect the Secret.**

```bash
kubectl get secret db-secret
kubectl describe secret db-secret
kubectl get secret db-secret -o yaml
```

Notice the values are base64-encoded.

**3. Decode a value manually.**

```bash
echo "c3VwZXJzZWNyZXQxMjM=" | base64 --decode
```

**4. Inject as environment variables.**

```yaml
# pod-secret-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo DB_USER=$DB_USER && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_USER
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
```

```bash
kubectl apply -f pod-secret-env.yaml
kubectl logs pod-secret-env
```

**5. Mount as a volume.**

```yaml
# pod-secret-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-vol
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "ls /etc/secrets && cat /etc/secrets/DB_USER && sleep 3600"]
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
```

```bash
kubectl apply -f pod-secret-vol.yaml
kubectl logs pod-secret-vol
```

### What to Observe

- Are the Secret values visible in plain text inside the pod?
- What is the difference between base64 encoding and encryption?

---

## Cleanup

```bash
kubectl delete pod my-pod --ignore-not-found
kubectl delete pod sidecar-pod --ignore-not-found
kubectl delete deployment web-app --ignore-not-found
kubectl delete statefulset web-stateful --ignore-not-found
kubectl delete daemonset node-monitor --ignore-not-found
kubectl delete job hello-job --ignore-not-found
kubectl delete cronjob hello-cron --ignore-not-found
kubectl delete pod pod-config-env --ignore-not-found
kubectl delete pod pod-config-vol --ignore-not-found
kubectl delete pod pod-secret-env --ignore-not-found
kubectl delete pod pod-secret-vol --ignore-not-found
kubectl delete configmap app-config --ignore-not-found
kubectl delete secret db-secret --ignore-not-found
```

If you need to remove any YAML files you created locally, delete them from your working directory as well.

---

## Optional Stretch Tasks

### Stretch 1: Pod Failure Analysis

- Create a pod with a bad image name.
- Observe the resulting `ImagePullBackOff`.
- Use `kubectl describe pod` to identify the error.

### Stretch 2: Memory Limit Test

- Add a very small memory limit to a pod.
- Generate memory pressure inside the container.
- Observe the `OOMKilled` status.

### Stretch 3: CronJob History

- Change the CronJob schedule to run every minute.
- Observe the Job history over 5 minutes.
- Check how many completed Jobs are retained.

### Stretch 4: ConfigMap Update Behavior

- Update the ConfigMap value.
- Observe whether a running pod sees the change automatically.
- Compare env var injection vs mounted file behavior.

---

## Self-Check Questions

1. What is the key difference between a Deployment and a StatefulSet?
2. What workload type ensures one pod on every node?
3. What does `READY 2/2` mean in a pod listing?
4. Where would you look first if a pod is in `CrashLoopBackOff`?
5. What is the difference between a ConfigMap and a Secret in terms of security?
