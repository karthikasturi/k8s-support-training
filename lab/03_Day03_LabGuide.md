# Day 03 — Lab Guide: Helm Charts

**Session:** Helm Charts  
**Duration:** ~2.5 hours  
**Environment:** Pre-provisioned shared cluster  

---

## Overview

In this lab you will work with Helm from a support-engineering perspective. You will install a chart, inspect the release, change values, upgrade the release, and roll back a bad version. You will also use Helm commands to render templates and debug chart issues.

### What You Will Learn

- How Helm charts are structured
- How releases and revisions work
- How to override values safely
- How to inspect release state and history
- How to debug rendered templates before installing

### Before You Begin

- You have completed Day 01 and Day 02 labs
- `kubectl` is connected to the cluster
- Helm is installed on your machine
- You are working in your assigned namespace

---

## Lab 1: Install a Helm Chart

### Summary
Install a simple Helm chart and verify that it created a release in the cluster.

### Steps

**1. Add a chart repository.**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**2. Search for a chart.**

```bash
helm search repo nginx
```

**3. Install the chart.**

```bash
helm install my-nginx bitnami/nginx --namespace default
```

If the namespace does not exist or you are using a custom namespace, add `--create-namespace`.

**4. Confirm the release.**

```bash
helm list
helm status my-nginx
```

**5. Check the Kubernetes objects.**

```bash
kubectl get pods,svc -l app.kubernetes.io/instance=my-nginx
```

### What to Observe

- What is the release name?
- What Kubernetes resources did Helm create?
- What is the status shown by `helm status`?

---

## Lab 2: Understand the Chart Structure

### Summary
Inspect the chart files so you know where metadata, defaults, and templates live.

### Steps

**1. Pull the chart locally.**

```bash
helm pull bitnami/nginx --untar
ls -R nginx/
```

**2. Open the key files.**

Look for:
- `Chart.yaml`
- `values.yaml`
- `templates/`
- `_helpers.tpl`
- `NOTES.txt`

**3. Read the metadata.**

```bash
cat nginx/Chart.yaml
```

**4. Read the default values.**

```bash
cat nginx/values.yaml
```

**5. Render one template locally.**

```bash
helm template my-nginx ./nginx
```

### What to Observe

- Which file contains the chart version?
- Which file contains the default image tag?
- How many templates are included in the chart?

---

## Lab 3: Inspect the Release Lifecycle

### Summary
Look at the release history and understand how Helm tracks revisions.

### Steps

**1. View the release history.**

```bash
helm history my-nginx
```

**2. Check the current values in use.**

```bash
helm get values my-nginx
```

**3. View all values, including defaults.**

```bash
helm get values my-nginx --all
```

**4. Inspect the rendered manifests.**

```bash
helm get manifest my-nginx
```

**5. Describe the release status.**

```bash
helm status my-nginx
```

### What to Observe

- What revision is currently deployed?
- What is the difference between `helm get values` and `helm get values --all`?
- What Kubernetes objects are part of the release?

---

## Lab 4: Override Values

### Summary
Change the application settings by overriding chart values.

### Steps

**1. Inspect the default values for the chart.**

```bash
helm show values bitnami/nginx | head -40
```

**2. Create a custom values file.**

```yaml
# custom-values.yaml
service:
  type: ClusterIP
replicaCount: 2
image:
  tag: latest
```

**3. Upgrade the release using the custom file.**

```bash
helm upgrade my-nginx bitnami/nginx -f custom-values.yaml
```

**4. Override one value inline.**

```bash
helm upgrade my-nginx bitnami/nginx -f custom-values.yaml --set replicaCount=3
```

**5. Verify the update.**

```bash
helm get values my-nginx
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

### What to Observe

- Which value source has higher priority: `-f` or `--set`?
- Did the replica count change?
- Did the pod count change after the upgrade?

---

## Lab 5: Debug Rendered Templates

### Summary
Use Helm debugging commands to inspect the rendered YAML before deployment.

### Steps

**1. Lint the chart.**

```bash
helm lint ./nginx
```

**2. Render the chart locally.**

```bash
helm template my-nginx ./nginx
```

**3. Render with debug output.**

```bash
helm template my-nginx ./nginx --debug
```

**4. Run a dry-run install.**

```bash
helm install my-nginx-test ./nginx --dry-run --debug
```

**5. Search for a specific field in the rendered output.**

```bash
helm template my-nginx ./nginx | grep -n "image:"
```

### What to Observe

- Does `helm lint` report any issues?
- What rendered YAML does Helm produce?
- How is `--dry-run --debug` different from a real install?

---

## Lab 6: Upgrade and Roll Back

### Summary
Simulate a bad upgrade and then restore the previous working revision.

### Steps

**1. Create a deliberate change.**

```bash
helm upgrade my-nginx bitnami/nginx --set image.tag=badtag
```

**2. Observe the failure.**

```bash
helm status my-nginx
kubectl get pods -l app.kubernetes.io/instance=my-nginx
kubectl describe pod <pod-name>
```

**3. Check revision history.**

```bash
helm history my-nginx
```

**4. Roll back to the previous working revision.**

```bash
helm rollback my-nginx 1
```

**5. Verify the rollback.**

```bash
helm history my-nginx
helm status my-nginx
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

### What to Observe

- Which revision is now active?
- What does the history show after rollback?
- Did the pod status return to healthy?

---

## Lab 7: Release Inspection

### Summary
Use Helm inspection commands to understand what is deployed and how it was configured.

### Steps

**1. List all releases in the namespace.**

```bash
helm list -n default
```

**2. Show the release notes.**

```bash
helm get notes my-nginx
```

**3. Get the current release manifest.**

```bash
helm get manifest my-nginx
```

**4. Get the current applied values.**

```bash
helm get values my-nginx
```

**5. Compare with all values.**

```bash
helm get values my-nginx --all
```

### What to Observe

- What is the chart version?
- What namespace is the release deployed in?
- Which values were overridden from the default chart settings?

---

## Cleanup

```bash
helm uninstall my-nginx
helm uninstall my-nginx-test --ignore-not-found
helm repo remove bitnami
rm -rf nginx
rm -f custom-values.yaml
```

If you created any additional test releases, uninstall them as well.

---

## Optional Stretch Tasks

### Stretch 1: Multiple Values Files

- Create two values files with different replica counts.
- Install with both files and observe which one wins.
- Check the merged values using `helm get values --all`.

### Stretch 2: Failed Template Debugging

- Edit a copied chart and break one template on purpose.
- Run `helm lint` and `helm template` to identify the issue.
- Fix the template and rerun the commands.

### Stretch 3: Namespace-Based Release

- Install the same chart in a new namespace.
- Compare the release names and history across namespaces.
- Inspect how `helm list -A` changes the view.

### Stretch 4: Manifest Comparison

- Render the chart with two different values sets.
- Compare the generated manifests.
- Identify which resource fields changed.

---

## Self-Check Questions

1. What is the difference between a Helm chart and a Helm release?
2. Which command shows the full revision history of a release?
3. What takes higher priority: `values.yaml` defaults or a `--set` flag?
4. Which command renders templates locally without installing anything?
5. What does a rollback actually do to the revision history?
