# Capstone Project — Starter App to Kubernetes Support Workflow

## Source Repository

The capstone starts from this repository. Clone it and use it as-is:

```bash
git clone https://github.com/karthikasturi/starter-app.git
cd starter-app
```

**Repository:** https://github.com/karthikasturi/starter-app

The repo begins at the **container / Docker Compose stage only** — no
Kubernetes manifests and no Helm chart. Its initial layout is:

```text
starter-app/
├── app.py               # Flask app: / , /health , /version + Redis visit counter
├── requirements.txt     # Flask, gunicorn, redis
├── Dockerfile           # python:3.12-slim, non-root, gunicorn on :8080
├── docker-compose.yml   # web + redis services, persistent redis-data volume
├── app.config.env       # non-sensitive config   -> becomes a ConfigMap
├── .env.example         # secret template          -> becomes a Secret
├── .gitignore
├── LICENSE
└── README.md
```

## Scope of the Starter App

The app is a small two-service stack defined in `docker-compose.yml`:

- **web** — a Flask service (gunicorn, port 8080, non-root user) with three
  routes:
  - `/` — greeting, app info, and a **visit counter** stored in Redis.
  - `/health` — readiness check; returns `200` only when Redis is reachable,
    otherwise `503`.
  - `/version` — app name and version.
- **redis** — a stateful key/value store that holds the visit counter on a
  **persistent volume** (`redis-data`).

Configuration is already split the way Kubernetes expects it:

| Source in repo                         | Contents                                                        | Maps to            |
| -------------------------------------- | -------------------------------------------------------------- | ------------------ |
| `app.config.env` (committed)           | `APP_NAME`, `APP_VERSION`, `GREETING`, `PORT`, `REDIS_HOST`, `REDIS_PORT` | **ConfigMap**      |
| `.env` (gitignored, from `.env.example`) | `REDIS_PASSWORD`                                              | **Secret**         |
| `redis-data` volume                    | Redis append-only data                                          | **StatefulSet + PVC** |

## Project Goal

Starting from this containerized Flask + Redis app, the support team turns it
into a Kubernetes application package — manifests, Helm packaging, networking,
scaling, storage, and operational troubleshooting — in lockstep with the
course sessions.

## Mapping to Course Sessions

There is **no Session 1 deliverable** — Session 1 (Architecture & Setup) is
covered by lecture and lab only. The capstone begins at Session 2, and each
session's tasks match that session's outline topics:

- **Session 2 — Kubernetes Workloads:** deploy the web app; externalize
  config and secrets; set resource requests/limits; create the core workload
  types (Deployment, **Job, CronJob, DaemonSet**).
- **Session 3 — Helm Charts:** package the manifests and operate the release.
- **Session 4 — Ingress, Egress & Networking:** Services and service types,
  the Redis Service, Ingress, routing, TLS, and a NetworkPolicy.
- **Session 5 — Scaling, Deployment Strategies & Storage:** scaling, HPA,
  rollout strategy, and the Redis **StatefulSet + PVC**.
- **Session 6 — Monitoring, Advanced Topics & Troubleshooting:** probes,
  monitoring signals, RBAC, quotas/limits, scheduling controls, and an
  incident-recovery drill.

## Conventions Used in This Capstone

So the tasks are unambiguous, use these names and values everywhere:

| Item                     | Value                                                        |
| ------------------------ | ----------------------------------------------------------- |
| Training namespace       | `capstone`                                                  |
| Image                    | `starter-app:1.0.0` (built from the repo `Dockerfile`)      |
| Web labels               | `app=web`                                                   |
| Redis labels             | `app=redis`                                                 |
| Web Deployment           | `web` (container port `8080`)                               |
| ConfigMap                | `app-config` (keys from `app.config.env`)                   |
| Secret                   | `redis-secret` (key `REDIS_PASSWORD`)                       |
| Web Service              | `web` (ClusterIP, port `8080` → targetPort `8080`)          |
| Redis Service            | `redis` (headless, `clusterIP: None`, port `6379`) — must equal `REDIS_HOST` |
| Redis StatefulSet        | `redis` (`serviceName: redis`, volume `data` at `/data`)    |
| Ingress                  | `web` (host `web.capstone.local`, path `/`)                 |
| HPA                      | `web`                                                       |
| NetworkPolicy            | `redis-allow-web`                                           |
| Job / CronJob            | `config-check`                                              |
| DaemonSet                | `node-logger`                                               |
| RBAC objects             | `support-readonly` (ServiceAccount + Role + RoleBinding)    |
| Governance               | `capstone-quota` (ResourceQuota), `capstone-limits` (LimitRange) |

Create the namespace once before Session 2 and run every command against it:

```bash
kubectl create namespace capstone
kubectl config set-context --current --namespace=capstone
```

Make the image available to your cluster after `docker build -t starter-app:1.0.0 .`:
Docker Desktop uses local images directly; for **kind** run
`kind load docker-image starter-app:1.0.0`; for **minikube** run
`minikube image load starter-app:1.0.0`.

## Task Format

Every task uses the same pattern:

- **Current state:** What exists before the task.
- **Action to perform:** What to create or change (exact, unambiguous).
- **Expected outcome:** What the learner should see, with a command to verify it.
- **Workload classification:** Which Kubernetes object or workload type is used.

---

## Session 2 — Kubernetes Workloads

*Outline focus: pods & containers, workload types, pod lifecycle, resource
requests/limits, ConfigMaps and Secrets.*

The web app becomes a Deployment, its configuration moves into a ConfigMap and
a Secret, resource requests/limits are set, and the core workload types
(Deployment, Job, CronJob, DaemonSet) are created and classified. Services and
Redis arrive later (Sessions 4 and 5), so reach the app here with
`kubectl port-forward deploy/web 8080:8080`.

### Task 2.1 — Deploy the web app as a Deployment

- **Current state:** The app exists only as the `starter-app:1.0.0` image and a
  Compose service; no Kubernetes objects exist in the `capstone` namespace.
- **Action to perform:** Create a **Deployment** named `web` with `replicas: 2`,
  label `app=web`, one container using image `starter-app:1.0.0` with
  `containerPort: 8080`.
- **Expected outcome:** Two pods are `Running` under one ReplicaSet.
  Verify: `kubectl get deploy web` shows `READY 2/2`, and `kubectl get rs,pods -l app=web` lists one ReplicaSet and two pods.
- **Workload classification:** **Deployment**.

### Task 2.2 — Inspect the pod lifecycle and self-healing

- **Current state:** The `web` Deployment is running with 2 pods.
- **Action to perform:** Describe one pod and read its phase transitions and
  events, then delete that pod with `kubectl delete pod <name>`.
- **Expected outcome:** The ReplicaSet immediately recreates the pod and the
  count returns to 2.
  Verify: `kubectl get pods -l app=web -w` shows a new pod replacing the deleted one; `kubectl describe pod <name>` shows the `Scheduled → Pulled → Created → Started` events.
- **Workload classification:** **Deployment** (pod lifecycle).

### Task 2.3 — Externalize non-sensitive config as a ConfigMap

- **Current state:** The container relies on the image's built-in defaults for
  `APP_NAME`, `GREETING`, etc.
- **Action to perform:** Create a ConfigMap `app-config` from the repo file
  (`kubectl create configmap app-config --from-env-file=app.config.env`) and
  inject it into the `web` container with `envFrom.configMapRef`. Roll the
  Deployment.
- **Expected outcome:** The app serves config from the ConfigMap. Change
  `GREETING` in the ConfigMap, restart the Deployment, and the `/` response
  changes.
  Verify: `kubectl exec deploy/web -- printenv GREETING` matches the ConfigMap; after `kubectl rollout restart deploy/web`, `curl` (via port-forward) shows the new greeting.
- **Workload classification:** **Deployment + ConfigMap**.

### Task 2.4 — Provide the Redis password as a Secret

- **Current state:** `REDIS_PASSWORD` is not present in any Kubernetes object.
- **Action to perform:** Create Secret `redis-secret` with key `REDIS_PASSWORD`
  (`kubectl create secret generic redis-secret --from-literal=REDIS_PASSWORD=<value>`)
  and inject it into the `web` container as env var `REDIS_PASSWORD` via
  `secretKeyRef`.
- **Expected outcome:** The password is delivered through the Secret, not
  plaintext in the image.
  Verify: `kubectl get secret redis-secret -o jsonpath='{.data.REDIS_PASSWORD}' | base64 -d` returns the value, and `kubectl exec deploy/web -- printenv REDIS_PASSWORD` matches it.
- **Workload classification:** **Deployment + Secret**.

### Task 2.5 — Set resource requests and limits

- **Current state:** The `web` container runs with no resource constraints
  (QoS `BestEffort`).
- **Action to perform:** Add to the `web` container: requests
  `cpu: 100m, memory: 128Mi` and limits `cpu: 500m, memory: 256Mi`.
- **Expected outcome:** The pod shows the requests/limits and a non-BestEffort
  QoS class (needed for the HPA in Session 5).
  Verify: `kubectl describe pod -l app=web` shows the Requests/Limits; `kubectl get pod -l app=web -o jsonpath='{.items[0].status.qosClass}'` returns `Burstable`.
- **Workload classification:** **Deployment** (resource management).

### Task 2.6 — Create a Job (workload type)

- **Current state:** Only the long-running `web` Deployment exists.
- **Action to perform:** Create a **Job** `config-check` that runs the
  `starter-app:1.0.0` image once with command
  `sh -c 'echo "APP_NAME=$APP_NAME"; test -n "$REDIS_PASSWORD" && echo "secret OK"'`,
  pulling env from `app-config` (configMapRef) and `redis-secret` (secretKeyRef).
  It must not depend on Redis.
- **Expected outcome:** The Job runs to completion and its log proves config and
  secret are injected.
  Verify: `kubectl get job config-check` shows `COMPLETIONS 1/1`; `kubectl logs job/config-check` prints `APP_NAME=starter-app` and `secret OK`.
- **Workload classification:** **Job**.

### Task 2.7 — Create a CronJob (workload type)

- **Current state:** A one-time `config-check` Job has been demonstrated.
- **Action to perform:** Create a **CronJob** `config-check` with schedule
  `*/1 * * * *` running the same container as Task 2.6.
- **Expected outcome:** A new Job (and pod) is created every minute and
  completes.
  Verify: `kubectl get cronjob config-check` shows a recent `LAST SCHEDULE`; `kubectl get jobs -l job-name` lists one Job per minute, each `Complete`.
- **Workload classification:** **CronJob**.

### Task 2.8 — Create a DaemonSet (workload type)

- **Current state:** Workloads so far are Deployment, Job, and CronJob.
- **Action to perform:** Create a **DaemonSet** `node-logger` that runs one
  `busybox` pod per node with command
  `sh -c 'while true; do echo "$(date) node=$NODE_NAME alive"; sleep 3600; done'`,
  exposing `NODE_NAME` via the downward API (`spec.nodeName`).
- **Expected outcome:** Exactly one `node-logger` pod runs on each schedulable
  node.
  Verify: `kubectl get ds node-logger` shows `DESIRED = CURRENT = READY = <node count>`; `kubectl get pods -l app=node-logger -o wide` shows one pod per node.
- **Workload classification:** **DaemonSet**.

### Task 2.9 — Diagnose one controlled failure

- **Current state:** All Session 2 workloads are healthy.
- **Action to perform:** Break the `web` Deployment in exactly one way — set the
  image to a non-existent tag `starter-app:does-not-exist` — apply it, diagnose,
  then restore the correct image.
- **Expected outcome:** The new pods fail and the learner identifies the cause
  from Kubernetes signals, then recovers.
  Verify: `kubectl get pods -l app=web` shows `ImagePullBackOff`/`ErrImagePull`; `kubectl describe pod <name>` shows the failed pull event; after restoring the tag, `kubectl rollout status deploy/web` reports success.
- **Workload classification:** **Deployment** (failure analysis).

---

## Session 3 — Add Helm Packaging

*Outline focus: Helm basics, chart structure, release lifecycle, value
overrides, debugging rendered templates.*

### Task 3.1 — Convert the manifests into a Helm chart

- **Current state:** Plain manifests exist for the `web` Deployment, `app-config`
  ConfigMap, and `redis-secret` Secret.
- **Action to perform:** Run `helm create starter-app`, strip the boilerplate,
  and add templates for the Deployment, ConfigMap, and Secret (with
  `_helpers.tpl`, `Chart.yaml`, `values.yaml`).
- **Expected outcome:** `helm template starter-app ./starter-app` renders valid
  Deployment, ConfigMap, and Secret YAML with no errors.
- **Workload classification:** **Helm-managed Deployment + ConfigMap + Secret**.

### Task 3.2 — Make the chart value-driven

- **Current state:** The templates contain hardcoded values.
- **Action to perform:** Move image repository/tag, `replicaCount`, resource
  requests/limits, and the app settings into `values.yaml`, and reference them in
  the templates.
- **Expected outcome:** Overriding a value changes the rendered output.
  Verify: `helm template starter-app ./starter-app --set replicaCount=3` shows `replicas: 3` in the Deployment.
- **Workload classification:** **Helm release configuration**.

### Task 3.3 — Install the first release

- **Current state:** The chart renders but is not installed.
- **Action to perform:** Run
  `helm install starter-app ./starter-app -n capstone`.
- **Expected outcome:** The release is `deployed` and the app pods run from it.
  Verify: `helm list -n capstone` shows `starter-app` with status `deployed`; `kubectl get deploy web` shows the pods.
- **Workload classification:** **Helm release**.

### Task 3.4 — Inspect release state, history, and rendered manifests

- **Current state:** The release is installed and stable.
- **Action to perform:** Run `helm status`, `helm history`, `helm get values`,
  and `helm get manifest` for the release.
- **Expected outcome:** The learner can state exactly what Helm deployed and at
  which revision.
  Verify: `helm history starter-app -n capstone` shows revision 1 `deployed`; `helm get manifest starter-app -n capstone` matches the live objects.
- **Workload classification:** **Helm release inspection**.

### Task 3.5 — Upgrade and rollback

- **Current state:** A working release at revision 1 exists.
- **Action to perform:** Upgrade with a valid change
  (`helm upgrade ... --set replicaCount=3`), then perform a bad upgrade
  (`--set image.tag=does-not-exist`), then `helm rollback starter-app 1`.
- **Expected outcome:** The learner traces revisions and recovers the app.
  Verify: `helm history` shows revisions 1–3; after rollback, `kubectl rollout status deploy/web` succeeds and `helm list` shows status `deployed`.
- **Workload classification:** **Helm release lifecycle**.

---

## Session 4 — Ingress, Egress & Networking

*Outline focus: service types, Ingress and ingress controller, routing rules,
TLS handling, network policies, cluster networking.*

### Task 4.1 — Expose the web app with a Service and compare service types

- **Current state:** The app is reachable only via `kubectl port-forward`.
- **Action to perform:** Create a **ClusterIP** Service `web` selecting `app=web`
  on port `8080`. Then temporarily change the type to `NodePort` and inspect the
  assigned node port, and review when `LoadBalancer` would be used.
- **Expected outcome:** The app is reachable by Service name in-cluster, and the
  learner can explain ClusterIP vs NodePort vs LoadBalancer.
  Verify: `kubectl get svc web` shows the ClusterIP and port `8080`; `kubectl run tmp --rm -it --image=busybox -- wget -qO- web:8080/version` returns the version JSON.
- **Workload classification:** **Service**.

### Task 4.2 — Create the headless Redis Service

- **Current state:** No backend Service exists; Redis pods arrive in Session 5.
- **Action to perform:** Create a **headless Service** named `redis`
  (`clusterIP: None`) selecting `app=redis` on port `6379`. The name must equal
  `REDIS_HOST`.
- **Expected outcome:** A stable DNS name `redis.capstone.svc.cluster.local`
  exists, ready for the StatefulSet (no endpoints yet — expected).
  Verify: `kubectl get svc redis` shows `CLUSTER-IP None`; `kubectl get endpoints redis` is empty for now.
- **Workload classification:** **Service (headless)**.

### Task 4.3 — Add an Ingress with a routing rule

- **Current state:** The web app is reachable only inside the cluster. An ingress
  controller is installed in the cluster.
- **Action to perform:** Create an **Ingress** `web` routing host
  `web.capstone.local`, path `/`, to Service `web` on port `8080`.
- **Expected outcome:** The app is reachable through the Ingress host.
  Verify: with the host mapped to the ingress IP, `curl http://web.capstone.local/version` returns the version JSON; `kubectl describe ingress web` shows the backend.
- **Workload classification:** **Ingress + Service**.

### Task 4.4 — Enable TLS on the Ingress

- **Current state:** The Ingress serves HTTP only.
- **Action to perform:** Create a TLS Secret (cert + key) for
  `web.capstone.local` and reference it in the Ingress `tls` block.
- **Expected outcome:** The app is reachable over HTTPS with TLS terminated at
  the Ingress.
  Verify: `curl -k https://web.capstone.local/version` returns the version JSON; `kubectl get ingress web -o jsonpath='{.spec.tls}'` shows the TLS Secret.
- **Workload classification:** **Ingress** (TLS termination).

### Task 4.5 — Restrict traffic with a NetworkPolicy

- **Current state:** All pod-to-pod traffic is allowed by default.
- **Action to perform:** Create a NetworkPolicy `redis-allow-web` that allows
  ingress to `app=redis` on `6379` **only** from pods labelled `app=web`, and
  denies other sources.
- **Expected outcome:** `web` can reach Redis; an unrelated pod cannot.
  Verify (after Session 5 Redis exists, or with a temporary `app=redis` pod): a `busybox` pod **without** `app=web` fails `nc -zv redis 6379`, while `kubectl exec deploy/web -- nc -zv redis 6379` succeeds.
- **Workload classification:** **NetworkPolicy**.

### Task 4.6 — Troubleshoot a broken route

- **Current state:** The route works end to end.
- **Action to perform:** Break it in exactly one way — change the `web` Service
  selector to `app=web-bad` — observe, then restore it.
- **Expected outcome:** The learner traces the failure Ingress → Service →
  Endpoints → Pod and finds the empty endpoint list.
  Verify: while broken, `kubectl get endpoints web` is empty and the Ingress host returns `503`; after restoring the selector, endpoints repopulate and the route works.
- **Workload classification:** **Ingress + Service + Deployment**.

---

## Session 5 — Scaling, Deployment Strategies & Storage

*Outline focus: pod scaling, autoscaling, deployment strategies, rollout
behavior, storage basics, persistent volumes and claims.*

### Task 5.1 — Scale the web Deployment manually

- **Current state:** `web` runs with a fixed replica count.
- **Action to perform:** Run `kubectl scale deploy/web --replicas=4`, then back
  to `2`.
- **Expected outcome:** The replica count and pod count track the change.
  Verify: `kubectl get deploy web` shows `READY 4/4` then `2/2`; `kubectl get pods -l app=web` reflects the count.
- **Workload classification:** **Deployment**.

### Task 5.2 — Add a HorizontalPodAutoscaler

- **Current state:** `web` has resource requests (Task 2.5) but no autoscaler;
  metrics-server is installed.
- **Action to perform:** Create an HPA `web` targeting `web`, `minReplicas: 2`,
  `maxReplicas: 5`, `averageUtilization: 50` on CPU.
- **Expected outcome:** The HPA reads metrics and can scale between its bounds.
  Verify: `kubectl get hpa web` shows current/target CPU and a known replica count (not `<unknown>` once metrics flow).
- **Workload classification:** **Deployment + HPA**.

### Task 5.3 — Tune the deployment strategy and watch a rollout

- **Current state:** `web` uses the default RollingUpdate settings.
- **Action to perform:** Set `strategy.rollingUpdate` to
  `maxSurge: 1, maxUnavailable: 0`, trigger an update
  (`kubectl set image deploy/web web=starter-app:1.0.0 --record` or a label
  change), and watch the rollout. Practice `kubectl rollout pause/resume/undo`.
- **Expected outcome:** The rollout proceeds one surge pod at a time with zero
  unavailable, and the learner can pause/resume/undo it.
  Verify: `kubectl rollout status deploy/web` shows incremental progress; `kubectl rollout history deploy/web` lists the revisions.
- **Workload classification:** **Deployment** (rollout strategy).

### Task 5.4 — Add the Redis StatefulSet with persistent storage

- **Current state:** The headless `redis` Service exists (Task 4.2) but has no
  pods, so the app backend is unreachable.
- **Action to perform:** Create a **StatefulSet** `redis` with `serviceName: redis`,
  1 replica, image `redis:7-alpine`, command requiring auth from `redis-secret`,
  and a `volumeClaimTemplate` named `data` (e.g. `1Gi`) mounted at `/data`.
- **Expected outcome:** Redis gets a stable pod `redis-0` and a bound PVC; the
  web app now reaches Redis.
  Verify: `kubectl get statefulset redis` shows `READY 1/1`; `kubectl get pvc` shows `data-redis-0` `Bound`; via port-forward, `curl /` returns increasing `visits` and `/health` returns `200`.
- **Workload classification:** **StatefulSet + PersistentVolumeClaim**.

### Task 5.5 — Verify persistence across pod restarts

- **Current state:** Redis runs as a StatefulSet with a bound PVC.
- **Action to perform:** Hit `/` several times, note the counter, delete pod
  `redis-0`, let the StatefulSet recreate it, and hit `/` again.
- **Expected outcome:** The visit counter continues from its previous value —
  proving data lives on the PVC, not the pod.
  Verify: counter before delete = N; after `kubectl delete pod redis-0` and recreation, `curl /` shows `> N`; `kubectl get pvc data-redis-0` is still `Bound` to the same volume.
- **Workload classification:** **StatefulSet + PVC** (storage verification).

---

## Session 6 — Monitoring, Advanced Topics & Troubleshooting

*Outline focus: monitoring stack, metrics and alerts, RBAC basics, quotas and
limits, scheduling controls, troubleshooting flow.*

### Task 6.1 — Add liveness and readiness probes

- **Current state:** The app is fully functional (Session 5) but has no probes.
- **Action to perform:** Add to the `web` container an HTTP **liveness** and
  **readiness** probe on `GET /health` port `8080` (sensible
  `initialDelaySeconds`/`periodSeconds`).
- **Expected outcome:** Readiness gates traffic on Redis availability; stopping
  Redis makes pods `NotReady` and removes them from Service endpoints.
  Verify: `kubectl get pods -l app=web` shows `READY 1/1`; scale Redis to 0 and `kubectl get endpoints web` empties; restore Redis and it repopulates.
- **Workload classification:** **Deployment** (health probes).

### Task 6.2 — Surface monitoring and visibility signals

- **Current state:** Logs and status exist; operational visibility is limited.
- **Action to perform:** Use `kubectl top pods`/`nodes` (metrics-server) and the
  `/health` signal to identify the basic health signals, and document which ones
  a support engineer should watch and alert on.
- **Expected outcome:** The learner connects Kubernetes objects to operational
  signals.
  Verify: `kubectl top pods -l app=web` returns CPU/memory; `/health` returns `200` while healthy and `503` when Redis is down.
- **Workload classification:** **Deployment + observability layer**.

### Task 6.3 — Apply RBAC for a support user

- **Current state:** Access is not scoped for support operators.
- **Action to perform:** Create ServiceAccount `support-readonly`, a Role
  `support-readonly` allowing `get/list/watch` on `pods`, `pods/log`, and
  `events`, and a matching RoleBinding — all in `capstone`.
- **Expected outcome:** The support identity can read but not modify workloads.
  Verify: `kubectl auth can-i get pods --as=system:serviceaccount:capstone:support-readonly -n capstone` returns `yes`; `... delete pods ...` returns `no`.
- **Workload classification:** **RBAC (ServiceAccount + Role + RoleBinding)**.

### Task 6.4 — Apply quotas and limits to the namespace

- **Current state:** The `capstone` namespace has no governance controls.
- **Action to perform:** Apply a **ResourceQuota** `capstone-quota` (cap pods and
  total CPU/memory requests/limits) and a **LimitRange** `capstone-limits`
  (default container request/limit).
- **Expected outcome:** Quota usage is tracked and over-quota pods are rejected;
  containers without requests get defaults.
  Verify: `kubectl describe quota capstone-quota` shows used vs hard; creating pods beyond the cap fails with a quota error.
- **Workload classification:** **ResourceQuota + LimitRange**.

### Task 6.5 — Influence scheduling

- **Current state:** Pods are scheduled with defaults.
- **Action to perform:** Label a node and add a `nodeSelector` (or node affinity)
  to `web`; then taint a node and add a matching toleration, and observe
  placement.
- **Expected outcome:** Pods land only where the rules allow.
  Verify: `kubectl get pods -l app=web -o wide` shows pods on the selected/tolerated node(s); removing the toleration on a tainted node leaves pods `Pending` with a scheduling event.
- **Workload classification:** **Deployment** (scheduling controls).

### Task 6.6 — Run a controlled incident and recover

- **Current state:** The app is stable after hardening.
- **Action to perform:** Trigger one controlled issue via Helm
  (`helm upgrade ... --set image.tag=does-not-exist`), diagnose with logs,
  events, and `describe`, then `helm rollback` to the last good revision.
- **Expected outcome:** The learner completes the full support workflow: detect,
  diagnose, recover, verify.
  Verify: during the incident `kubectl get pods` shows `ImagePullBackOff`; after `helm rollback`, `kubectl rollout status deploy/web` succeeds and `/health` returns `200`.
- **Workload classification:** **Helm release + Deployment**.

### Task 6.7 — Final operational review

- **Current state:** All earlier tasks are complete.
- **Action to perform:** Walk through the whole release — workloads, networking,
  config/secrets, storage, scaling, governance, RBAC — and explain the lifecycle
  from raw container to supported Kubernetes release.
- **Expected outcome:** The learner can explain the end-to-end lifecycle and
  where each support skill applies.
  Verify: `kubectl get all,cm,secret,ingress,netpol,hpa,pvc,quota,limitrange -n capstone` enumerates every object the capstone produced.
- **Workload classification:** **All relevant workload types used in the project**.

---

## Final Repository Layout After Completion

```text
starter-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── app.config.env
├── .env.example
├── .gitignore
├── LICENSE
├── README.md
├── k8s/
│   ├── deployment.yaml          # web (Session 2)
│   ├── configmap.yaml           # app-config (Session 2)
│   ├── secret.yaml              # redis-secret (Session 2)
│   ├── job.yaml                 # config-check (Session 2)
│   ├── cronjob.yaml             # config-check (Session 2)
│   ├── daemonset.yaml           # node-logger (Session 2)
│   ├── service.yaml             # web (Session 4)
│   ├── redis-service.yaml       # redis headless (Session 4)
│   ├── ingress.yaml             # web (Session 4)
│   ├── networkpolicy.yaml       # redis-allow-web (Session 4)
│   ├── hpa.yaml                 # web (Session 5)
│   ├── statefulset.yaml         # redis + PVC (Session 5)
│   ├── rbac.yaml                # support-readonly (Session 6)
│   ├── resourcequota.yaml       # capstone-quota (Session 6)
│   └── limitrange.yaml          # capstone-limits (Session 6)
└── helm/
    └── starter-app/
        ├── Chart.yaml
        ├── values.yaml
        ├── .helmignore
        ├── README.md
        └── templates/
            ├── _helpers.tpl
            ├── deployment.yaml
            ├── configmap.yaml
            ├── secret.yaml
            ├── service.yaml
            ├── redis-service.yaml
            ├── ingress.yaml
            ├── networkpolicy.yaml
            ├── hpa.yaml
            ├── statefulset.yaml
            ├── job.yaml
            ├── cronjob.yaml
            └── daemonset.yaml
```

## Minimal Capstone Boundaries

- One web Flask app, managed as one **Deployment**.
- One **Redis StatefulSet** (the only stateful component) with one **PVC**.
- One Helm chart.
- Services for the web app and for Redis; one **Ingress** route.
- One **ConfigMap** (`app-config`) and one **Secret** (`redis-secret`).
- **Job**, **CronJob**, and **DaemonSet** as workload-type demonstrations
  (Session 2).
- **HPA**, **NetworkPolicy**, **RBAC**, **ResourceQuota/LimitRange**, and health
  **probes** as scaling/networking/hardening tasks (Sessions 4–6).

## What Not to Add

- Multiple microservices.
- Business workflows.
- Complex database schema design.
- CI/CD pipeline implementation.
- Application development tasks unrelated to Kubernetes support.

## Final Result

By the end of Day 6, learners take a container-ready Flask + Redis application
from "no Kubernetes config at all" to a hardened, Helm-managed Kubernetes
release — with Services, Ingress, scaling, persistent storage, governance, and
operational troubleshooting experience, and a clear, support-oriented
understanding of each workload type.
