# aks-real-world

# Real-World AKS Application Platform

## Overview

This project builds a more production-style application platform using **Azure Kubernetes Service (AKS)**.

The goal is to practice important AKS operational concepts such as:

- Resource groups and AKS cluster creation
- Kubernetes namespaces
- Deployments and Services
- ConfigMaps
- Resource requests and limits
- Liveness and readiness probes
- Horizontal Pod Autoscaling
- Cluster Autoscaling
- Pod Disruption Budgets
- Rolling updates
- Rollbacks
- Troubleshooting

The application used in this project is a simple **NGINX web application**.

---

## Architecture

```text
Internet
   |
   v
Azure Load Balancer
   |
   v
Kubernetes Service
   |
   v
NGINX Deployment
   |
   +-- Pod 1
   +-- Pod 2
   +-- Pod 3
        |
        +-- Readiness Probe
        +-- Liveness Probe
        +-- CPU / Memory Limits
        |
        v
Horizontal Pod Autoscaler
        |
        v
Cluster Autoscaler
        |
        v
AKS Node Pool
```

---

## Technologies Used

- Microsoft Azure
- Azure Kubernetes Service
- Azure CLI
- Kubernetes
- kubectl
- NGINX
- YAML
- Horizontal Pod Autoscaler
- Cluster Autoscaler

---

## Project Structure

```text
aks-production-project/
│
├── namespace.yaml
├── configmap.yaml
├── deployment.yaml
├── service.yaml
├── hpa.yaml
├── pdb.yaml
└── README.md
```

---

# 1. Create Project Variables

```bash
RG2="rg-aks-production"
AKS2="aks-production"
LOCATION="canadacentral"
```

---

# 2. Create the Resource Group

```bash
az group create \
  --name "$RG2" \
  --location "$LOCATION"
```

Verify:

```bash
az group show \
  --name "$RG2" \
  --output table
```

---

# 3. Create the AKS Cluster

```bash
az aks create \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --node-count 2 \
  --generate-ssh-keys
```

Connect `kubectl` to the cluster:

```bash
az aks get-credentials \
  --resource-group "$RG2" \
  --name "$AKS2"
```

Verify:

```bash
kubectl get nodes
```

---

# 4. Create the Production Namespace

Create `namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: production
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespaces
```

---

# 5. Create the ConfigMap

Create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: web-config
  namespace: production

data:
  index.html: |
    <!DOCTYPE html>
    <html>
      <head>
        <title>AKS Production Project</title>
      </head>
      <body>
        <h1>AKS Production Application</h1>
        <p>Running on Azure Kubernetes Service.</p>
      </body>
    </html>
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

---

# 6. Create the Deployment

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web
  namespace: production

spec:
  replicas: 3

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: web

  template:

    metadata:
      labels:
        app: web

    spec:

      containers:

        - name: web

          image: nginx:1.28-alpine

          ports:
            - containerPort: 80

          resources:

            requests:
              cpu: "25m"
              memory: "32Mi"

            limits:
              cpu: "200m"
              memory: "128Mi"

          livenessProbe:

            httpGet:
              path: /
              port: 80

            initialDelaySeconds: 5
            periodSeconds: 10

          readinessProbe:

            httpGet:
              path: /
              port: 80

            initialDelaySeconds: 3
            periodSeconds: 5

          volumeMounts:

            - name: web-content
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html

      volumes:

        - name: web-content

          configMap:
            name: web-config
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get pods \
  -n production \
  -o wide
```

---

# 7. Create the LoadBalancer Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: web-service
  namespace: production

spec:

  type: LoadBalancer

  selector:
    app: web

  ports:

    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check the external IP:

```bash
kubectl get svc -n production
```

Open:

```text
http://<EXTERNAL-IP>
```

---

# 8. Create the Pod Disruption Budget

Create `pdb.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: web-pdb
  namespace: production

spec:

  minAvailable: 2

  selector:
    matchLabels:
      app: web
```

Apply:

```bash
kubectl apply -f pdb.yaml
```

Verify:

```bash
kubectl get pdb -n production
```

---

# 9. Configure Horizontal Pod Autoscaling

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: web-hpa
  namespace: production

spec:

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web

  minReplicas: 2
  maxReplicas: 8

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:
          type: Utilization
          averageUtilization: 50
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

Check:

```bash
kubectl get hpa -n production
```

---

# 10. Generate Test Traffic

Create a load generator:

```bash
kubectl run load-generator \
  --namespace production \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web-service > /dev/null; done"
```

Watch the autoscaler:

```bash
kubectl get hpa \
  -n production \
  -w
```

Watch pods:

```bash
kubectl get pods \
  -n production \
  -w
```

Delete the load generator when finished:

```bash
kubectl delete pod load-generator \
  -n production
```

---

# 11. Configure Cluster Autoscaling

The Cluster Autoscaler allows AKS to automatically add or remove worker nodes depending on workload requirements.

Typical configuration:

```bash
az aks update \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3
```

In this project, an Azure CLI API-version issue required using the AKS node-pool resource directly.

Example configuration:

```bash
SUB_ID=$(az account show --query id -o tsv)

NODEPOOL="nodepool1"

POOL_ID="/subscriptions/$SUB_ID/resourceGroups/$RG2/providers/Microsoft.ContainerService/managedClusters/$AKS2/agentPools/$NODEPOOL"
```

When using Git Bash on Windows:

```bash
export MSYS_NO_PATHCONV=1
```

Enable autoscaling:

```bash
az resource update \
  --ids "$POOL_ID" \
  --api-version "2026-06-01" \
  --set \
    properties.enableAutoScaling=true \
    properties.minCount=1 \
    properties.maxCount=3
```

Verify:

```bash
az resource show \
  --ids "$POOL_ID" \
  --api-version "2026-06-01" \
  --query "{
    Name:name,
    CurrentNodes:properties.count,
    Autoscaling:properties.enableAutoScaling,
    Minimum:properties.minCount,
    Maximum:properties.maxCount
  }" \
  --output table
```

---

# 12. Perform a Rolling Update

Update the NGINX image:

```bash
kubectl set image deployment/web \
  web=nginx:1.29-alpine \
  --namespace production
```

Check rollout status:

```bash
kubectl rollout status deployment/web \
  --namespace production
```

View rollout history:

```bash
kubectl rollout history deployment/web \
  --namespace production
```

---

# 13. Test a Failed Deployment

Deploy an invalid image:

```bash
kubectl set image deployment/web \
  web=nginx:this-version-does-not-exist \
  --namespace production
```

Check:

```bash
kubectl get pods -n production
```

You may see:

```text
ImagePullBackOff
```

Inspect the problem:

```bash
kubectl describe pod <POD-NAME> \
  -n production
```

Rollback:

```bash
kubectl rollout undo deployment/web \
  -n production
```

Verify:

```bash
kubectl get pods -n production
```

---

# Troubleshooting Commands

Check pods:

```bash
kubectl get pods -n production
```

Inspect a pod:

```bash
kubectl describe pod <POD-NAME> \
  -n production
```

View logs:

```bash
kubectl logs <POD-NAME> \
  -n production
```

Check recent events:

```bash
kubectl get events \
  -n production \
  --sort-by=.metadata.creationTimestamp
```

Check services:

```bash
kubectl get svc -n production
```

Check HPA:

```bash
kubectl describe hpa web-hpa \
  -n production
```

Check CPU and memory usage:

```bash
kubectl top pods -n production
```

```bash
kubectl top nodes
```

---

# Key Concepts Learned

## Readiness Probe

Determines whether a Pod is ready to receive network traffic.

## Liveness Probe

Determines whether a container is healthy. Kubernetes can restart containers that repeatedly fail the liveness check.

## Resource Requests

Specify the amount of CPU and memory Kubernetes should reserve for a Pod.

## Resource Limits

Define the maximum amount of CPU and memory a container can consume.

## Horizontal Pod Autoscaler

Automatically increases or decreases the number of Pods based on application load.

```text
High CPU
   |
   v
HPA
   |
   v
More Pods
```

## Cluster Autoscaler

Automatically changes the number of AKS worker nodes when more or less cluster capacity is required.

```text
More Pods
   |
   v
Insufficient Node Capacity
   |
   v
Cluster Autoscaler
   |
   v
Additional AKS Node
```

## Pod Disruption Budget

Helps maintain application availability during planned Kubernetes disruptions.

## Rolling Update

Gradually replaces old application Pods with a newer application version.

## Rollback

Restores a previous working Deployment if a new release fails.

---

# Project Outcome

This project provided hands-on experience operating a more production-style workload on **Azure Kubernetes Service**.

The complete workflow was:

```text
Azure Resource Group
        |
        v
AKS Cluster
        |
        v
Production Namespace
        |
        v
ConfigMap
        |
        v
Deployment
        |
        v
NGINX Pods
        |
        v
LoadBalancer Service
        |
        v
Internet
```

The project also added:

```text
Health Probes
Resource Requests / Limits
Pod Disruption Budget
Horizontal Pod Autoscaler
Cluster Autoscaler
Rolling Updates
Rollback
Troubleshooting
```

---

# Cleanup

Delete the resource group when the project is complete:

```bash
az group delete \
  --name "$RG2" \
  --yes \
  --no-wait
```

This removes the AKS cluster and the Azure resources created for this project.

---

# Summary

Through this project I gained practical experience with:

- AKS cluster creation
- Kubernetes workload deployment
- Application configuration
- Load balancing
- Application health checks
- Resource management
- Pod autoscaling
- Node autoscaling
- High availability concepts
- Rolling deployments
- Rollbacks
- AKS troubleshooting

This project demonstrates foundational operational skills required to manage containerized applications using **Azure Kubernetes Service**.