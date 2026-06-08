# 🚀 Zero-Downtime Blue-Green Deployments in Kubernetes

This repository provides a comprehensive guide, architectural overview, and the exact native Kubernetes manifest files required to implement a **Blue-Green Deployment** strategy. This release pattern completely decouples application deployment from traffic routing to achieve zero-downtime upgrades and rapid, risk-free rollbacks.

---

## 🧠 Core Concepts & Definitions

An **Application Release Model (ARM)** represents the workflow and method we use to safely transition our software code from a developer's machine into a live production environment so that end-users can seamlessly access the latest features. 

The **Blue-Green Deployment** model accomplishes this by maintaining two completely isolated, identical environments running concurrently inside our cluster:

*   **Blue Environment (Deployment):** Runs the current, stable version of the code/application currently handling live production traffic (`v1`).
*   **Green Environment (Deployment):** Runs the latest code, updated application, or upcoming version (`v2`) waiting to be tested and promoted.

---

## 🏗️ Architectural Visual Flow

```text
       [ End-Users ]                       [ Testing Team ]
             │                                     │
             ▼ (Live Traffic)                      ▼ (Pre-Prod Traffic)
┌───────────────────────────┐         ┌───────────────────────────┐
│ Service: javawebapp-live  │         │ Service: javawebapp-prod  │
│ Selector: version=v1 ──┐  │         │ Selector: version=v2      │
└────────────────────────┼──┘         └─────────────┬─────────────┘
                         │                          │
                         ▼                          ▼
                ┌─────────────────┐        ┌─────────────────┐
                │    Blue Pods    │        │   Green Pods    │
                │  (version: v1)  │        │  (version: v2)  │
                └─────────────────┘        └─────────────────┘
```

## 🛠️ Step-by-Step Implementation Runbook 
Step 1: Deploy the Blue Baseline (v1)
Initially, our application runs exclusively inside the Blue environment. We deploy our pods and expose them to our users using a live network service. <br>
blue-deployment.yml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue-deployment
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: web-app
      version: v1
      color: blue
  template:
    metadata:
      labels:
        app: web-app
        version: v1
        color: blue
    spec:
      containers:
      - name: webapp-container
        image: argoproj/rollouts-demo:blue
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
```
live-service.yml
```bash
apiVersion: v1
kind: Service
metadata:
  name: webapp-live-service
spec:
  type: LoadBalancer
  selector:
    version: v1
  ports:
  - port: 80
    targetPort: 8080
```
Apply these initial configurations to your control plane:
```bash
kubectl apply -f blue-deployment.yml
kubectl apply -f live-service.yml

# Verify that the blue pods are up, running, and accessible
kubectl get pods -o wide
```
Step 2: Build & Deploy the Green Environment (v2)
When developers update the code, compile the code using build tools (e.g., Maven clean/package), package it into a new container layer using a Dockerfile, tag it as v2, and push it to Docker Hub.
Now, we deploy the Green pods alongside the Blue pods. Crucial rule: We do not modify the running Blue environment. <br>
green-deployment.yml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green-deployment
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: web-app
      version: v2
      color: green
  template:
    metadata:
      labels:
        app: web-app
        version: v2
        color: green
    spec:
      containers:
      - name: webapp-container
        image: argoproj/rollouts-demo:green
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```
Apply the updated application code to the cluster:
```bash
kubectl apply -f green-deployment.yml

# Ensure the green pods are initialized without touching active production users
kubectl get pods -o wide
```
Step 3: Expose Green Pods to the Testing Team
Before routing live production customers to our new code, we create an isolated, pre-production Service that targets only the new v2 pods so that our QA/testing teams can thoroughly validate the build.<br>
pre-prod-service.yml
```bash
apiVersion: v1
kind: Service
metadata:
  name: webapp-prod-service
spec:
  type: NodePort
  selector:
    version: v2
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31785
```
Deploy the testing endpoint:
```bash
kubectl apply -f pre-prod-service.yml
```
The testing team can now access the new build directly using the specific NodePort (port 31785) to verify that the application works exactly as expected.
<br>
Step 4: Execute the Live Traffic Cutover
Once the QA team confirms that the v2 application code is fully stable, we promote it to production. We do this in a fraction of a second by simply updating the label selector inside our live-facing production Service manifest.
<br>
Update your live-service.yml file to point to the new version label:
<br>
```bash
apiVersion: v1
kind: Service
metadata:
  name: webapp-live-service
spec:
  type: LoadBalancer
  selector:
    version: v2 # Switched from v1 to v2 to instantaneously swing traffic
  ports:
  - port: 80
    targetPort: 8080
```
Apply the updated live routing configuration:
<br>
```bash
kubectl apply -f live-service.yml
```
The Result: The moment this command executes, the Kubernetes internal load balancer shifts its backend routing targets. All incoming customer requests are transparently routed away from the Blue (v1) pods and sent directly to the Green (v2) pods with zero downtime.
<br>
## 🔄 Emergency Fallback Protocol
If any hidden application error escapes testing and manifests in production post-cutover, you can instantly execute a fallback rollback. Open your live-service.yml file, change the selector criteria back to version: v1, and apply it immediately:
<br>
```bash
kubectl apply -f live-service.yml
```
Traffic will safely swing back onto your untouched, verified Blue pods within seconds!

## 🎯 Key Learnings from this Architecture
Through implementing this deployment model, several core DevOps concepts were practically validated:
* **Decoupling Deployment from Release:** Uploading new code to servers (deployment) and giving users access to that code (release) are two completely separate operational phases.
* **The Power of Label Selectors:** Kubernetes Services act as highly dynamic, internal load balancers. Simply changing a single word in a metadata label (from `v1` to `v2`) dictates the flow of entire enterprise networks.
* **Rollback Confidence:** Knowing that the `v1` pods remain untouched during the deployment of `v2` completely removes the fear of catastrophic production outages.

## ⚠️ Common Pitfalls & Troubleshooting (What Can Go Wrong)
While practicing and implementing this model, there are several strict operational risks to monitor:

* **Typo/Spelling Errors in Selectors:** Kubernetes is strictly declarative. If your `live-service.yml` selector asks for `version: v2` but your pods are accidentally labeled `verison: v2` (typo), the service will find zero endpoints. Traffic will hit a dead end, causing a complete application outage.
* **Resource Starvation (OOMKilled):** Because Blue-Green requires running both versions simultaneously, your underlying worker nodes must have the CPU and RAM capacity to support double the normal workload. If they do not, the Linux kernel will terminate pods with `OOMKilled` (Out of Memory) errors, crashing the cluster.
* **Database Schema Conflicts:** If your new Green deployment (`v2`) requires a change to the database structure (e.g., deleting a column), doing so will instantly break the currently running Blue deployment (`v1`) which still relies on that old column. *Solution: Database changes must always be backwards-compatible during a Blue-Green deployment.*
* **Dangling Cloud Costs:** If an engineer forgets to delete the old Blue deployment after the Green deployment has been stable for a few days, those unused pods will sit idle indefinitely, consuming expensive cloud compute resources.























