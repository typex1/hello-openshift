# hello-openshift

---

# ArgoCD Tutorial: GitOps Deployment with Kubernetes

This tutorial will guide you through setting up ArgoCD to automatically deploy applications from a GitHub repository to your Kubernetes cluster.

## Prerequisites

- Access to a Kubernetes/OpenShift cluster
- `kubectl` CLI installed and configured
- A GitHub repository with Kubernetes manifests (we'll use: https://github.com/typex1/hello-openshift)

---

## Step 1: Verify ArgoCD Installation

First, check if ArgoCD is installed in your cluster:

```bash
kubectl get pods -n argocd
```

**Expected output:** You should see several ArgoCD pods running:
- `argocd-application-controller`
- `argocd-repo-server`
- `argocd-server`
- `argocd-dex-server`
- `argocd-redis`
- `argocd-notifications-controller`

**Check if all pods are running:**
```bash
kubectl get pods -n argocd | grep -v Running
```

If you see any pods in `CrashLoopBackOff` or `Error` state, they need to be fixed before proceeding.

---

## Step 2: Fix Common ArgoCD Issues (If Needed)

If the `argocd-repo-server` pod is in `CrashLoopBackOff`, check the logs:

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -c copyutil --tail=20
```

If you see `/bin/ln: Already exists` error, fix it with:

```bash
kubectl patch deployment argocd-repo-server -n argocd --type=json -p='[{"op": "replace", "path": "/spec/template/spec/initContainers/0/args/0", "value": "/bin/cp --update=none /usr/local/bin/argocd /var/run/argocd/argocd && /bin/ln -sf /var/run/argocd/argocd /var/run/argocd/argocd-cmp-server"}]'
```

Wait for the pod to restart and verify it's running:

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server
```

---

## Step 3: Verify ArgoCD CLI (Optional)

Check if the ArgoCD CLI is installed:

```bash
which argocd
```

If not installed, you can still use `kubectl` commands to manage ArgoCD applications.

---

## Step 4: Understand the GitHub Repository Structure

Our example repository (https://github.com/typex1/hello-openshift) contains:
- `deployment.yaml` - Kubernetes Deployment manifest
- `service.yaml` - Kubernetes Service manifest

These files define a simple web application that ArgoCD will deploy.

---

## Step 5: Create the ArgoCD Application Manifest

Create a file called `argocd-hello-openshift-app.yaml`:

```bash
cat > argocd-hello-openshift-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello-openshift
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/typex1/hello-openshift
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: hello-openshift
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

**Understanding the manifest:**

- `metadata.name`: Name of your ArgoCD application
- `source.repoURL`: Your GitHub repository URL
- `source.targetRevision`: Git branch to track (main)
- `source.path`: Directory in the repo containing manifests (`.` means root)
- `destination.namespace`: Target namespace for deployment
- `syncPolicy.automated`: Enables automatic sync when Git changes are detected
- `syncPolicy.automated.prune`: Removes resources deleted from Git
- `syncPolicy.automated.selfHeal`: Reverts manual changes to match Git state
- `syncOptions.CreateNamespace`: Creates the namespace if it doesn't exist

---

## Step 6: Deploy the ArgoCD Application

Apply the manifest to create the ArgoCD Application:

```bash
kubectl apply -f argocd-hello-openshift-app.yaml
```

**Expected output:**
```
application.argoproj.io/hello-openshift created
```

---

## Step 7: Monitor the Deployment

Check the ArgoCD Application status:

```bash
kubectl get application -n argocd hello-openshift
```

**Expected output:**
```
NAME              SYNC STATUS   HEALTH STATUS
hello-openshift   Synced        Healthy
```

**Status meanings:**
- `SYNC STATUS`:
  - `Synced` - Application matches Git state
  - `OutOfSync` - Differences between Git and cluster
- `HEALTH STATUS`:
  - `Healthy` - All resources running correctly
  - `Progressing` - Deployment in progress
  - `Degraded` - Issues with resources

---

## Step 8: Verify Deployed Resources

Check all resources in the target namespace:

```bash
kubectl get all -n hello-openshift
```

**Expected output:**
```
NAME                                   READY   STATUS    RESTARTS   AGE
pod/hello-openshift-xxxxx-xxxxx        1/1     Running   0          1m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/hello-openshift   ClusterIP   172.30.xxx.xxx   <none>        80/TCP    1m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-openshift   1/1     1            1           1m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-openshift-xxxxx        1         1         1       1m
```

**Verify each component:**

1. **Pod status:**
```bash
kubectl get pods -n hello-openshift
```
Should show `Running` and `1/1` Ready.

2. **Service details:**
```bash
kubectl get svc -n hello-openshift
```
Should show the ClusterIP and port 80.

3. **Deployment status:**
```bash
kubectl get deployment -n hello-openshift
```
Should show `1/1` replicas available.

---

## Step 9: Test the Application with curl

Test the service from within the cluster:

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never -- curl -s http://hello-openshift.hello-openshift.svc.cluster.local
```

**Expected output:**
```
Hello OpenShift! Version: 1.0.0
pod "curl-test" deleted
```

**Understanding the command:**
- `kubectl run curl-test` - Creates a temporary pod named curl-test
- `--image=curlimages/curl:latest` - Uses the curl container image
- `--rm` - Deletes the pod after execution
- `-i` - Shows output interactively
- `--restart=Never` - Runs as a one-time job
- `curl -s http://hello-openshift.hello-openshift.svc.cluster.local` - Makes HTTP request to the service

**Service DNS format:** `<service-name>.<namespace>.svc.cluster.local`

---

## Step 10: Trigger a Manual Sync

Even though auto-sync is enabled, you can manually trigger a sync:

```bash
kubectl patch app hello-openshift -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

Check the sync status:

```bash
kubectl get application hello-openshift -n argocd -o jsonpath='{.status.sync.status} - {.status.operationState.phase}'
```

**Expected output:**
```
Synced - Succeeded
```

---

## Step 11: Test After Manual Sync

Run the curl test again to verify the application is still working:

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never -- curl -s http://hello-openshift.hello-openshift.svc.cluster.local
```

**Note:** If the version number changes, it means ArgoCD synced a different version from Git.

---

## Step 12: Understanding ArgoCD Sync Frequency

By default, ArgoCD checks your Git repository for changes **every 3 minutes**.

**To check the current setting:**
```bash
kubectl get configmap argocd-cm -n argocd -o yaml | grep timeout.reconciliation
```

**To change the sync interval globally (e.g., to 1 minute):**
```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"60s"}}'
```

**Alternative: Use webhooks for instant updates**
- Configure GitHub webhooks to notify ArgoCD immediately on push
- This is faster than polling but requires network access from GitHub to your cluster

---

## Step 13: View Detailed Application Information

Get detailed information about your application:

```bash
kubectl describe application hello-openshift -n argocd
```

This shows:
- Source repository details
- Sync status and history
- Health status of all resources
- Recent events

---

## Step 14: Clean Up (Optional)

To remove the application and all deployed resources:

```bash
kubectl delete application hello-openshift -n argocd
```

This will:
1. Delete the ArgoCD Application
2. Remove all resources from the `hello-openshift` namespace (because `prune: true`)
3. Delete the namespace (if empty)

Verify cleanup:
```bash
kubectl get all -n hello-openshift
```

---

## Common Troubleshooting

### Application shows "OutOfSync"

**Check what's different:**
```bash
kubectl get application hello-openshift -n argocd -o yaml | grep -A 20 status
```

**Force sync:**
```bash
kubectl patch app hello-openshift -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### Application shows "Degraded"

**Check pod logs:**
```bash
kubectl logs -n hello-openshift -l app=hello-openshift
```

**Check pod events:**
```bash
kubectl describe pod -n hello-openshift -l app=hello-openshift
```

### Service not responding

**Verify service endpoints:**
```bash
kubectl get endpoints -n hello-openshift
```

**Check if pods are ready:**
```bash
kubectl get pods -n hello-openshift -o wide
```

---

## Key Concepts Summary

1. **GitOps**: Git is the single source of truth for your infrastructure
2. **Declarative**: You declare the desired state in Git, ArgoCD makes it happen
3. **Automated Sync**: ArgoCD continuously monitors Git and applies changes
4. **Self-Healing**: Manual changes to the cluster are automatically reverted
5. **Prune**: Resources deleted from Git are removed from the cluster

---

## Next Steps

### 1. Test GitOps in Action: Update the Application

The most important step to understand GitOps is to see it in action. Let's update the application by changing the container image version.

**Edit `deployment.yaml` in your GitHub repo:**

Change this line:
```yaml
image: fspiess31/hello-openshift:1.0.0
```

To this line:
```yaml
image: fspiess31/hello-openshift:1.0.1
```

**What happens next:**
1. Commit and push the change to GitHub
2. Within 3 minutes (or instantly with webhooks), ArgoCD detects the change
3. ArgoCD automatically pulls the new image version
4. The deployment is updated with zero manual intervention
5. Test with curl to see the new version:

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never -- curl -s http://hello-openshift.hello-openshift.svc.cluster.local
```

You should see: `Hello OpenShift! Version: 1.0.1`

**This demonstrates the core GitOps principle:** Git is the single source of truth. Change Git, and your cluster automatically reflects those changes.

### 2. Additional Learning Steps

- **Add more manifests**: Create additional Kubernetes resources in your repo
- **Explore ArgoCD UI**: Access the ArgoCD web interface for visual management
- **Set up webhooks**: Configure GitHub webhooks for instant sync
- **Create multiple applications**: Deploy different apps to different namespaces

---

## Additional Resources

- ArgoCD Documentation: https://argo-cd.readthedocs.io/
- GitOps Principles: https://opengitops.dev/
- Kubernetes Documentation: https://kubernetes.io/docs/

---

## Quick Reference Commands

```bash
# Check ArgoCD status
kubectl get pods -n argocd

# List all applications
kubectl get applications -n argocd

# Get application details
kubectl get application <app-name> -n argocd

# Trigger manual sync
kubectl patch app <app-name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'

# Check deployed resources
kubectl get all -n <namespace>

# Test service with curl
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never -- curl -s http://<service>.<namespace>.svc.cluster.local

# Delete application
kubectl delete application <app-name> -n argocd
```

---

**Congratulations!** You've successfully deployed an application using ArgoCD and GitOps principles.