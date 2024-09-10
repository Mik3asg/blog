---
title: "Kubernetes - Zero Downtime Deployments: Blue/Green Strategy"
date: 2024-09-09T16:49:36+01:00
draft: false
tags: ['Kubernetes', 'Deployment', 'Strategy', 'Service', 'Routing', 'Environment']
---
# Overview
In this guide, I'll demonstrate a blue-green deployment strategy in Kubernetes using Deployments and Services. The goal is to achieve zero downtime by running two sets of pods: the current version (v1.0, blue) and the new version (v2.0, green). I'll also explain how to roll back from green to blue if necessary.


## Setup the Blue Environment 
The blue-deployment.yaml defines the current environment running version 1.0 of the app.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-deploy
  labels:
    app: my-node-app
    env: blue
    version: v1.0
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-node-app
      env: blue
  template:
    metadata:
      labels:
        app: my-node-app
        env: blue
        version: v1.0
    spec:
      containers:
      - name: my-node-app
        image: mik3asg/k8s-zero-downtime-deployment:v1.0
        ports:
        - containerPort: 3000
```
### Deploy the Blue Environment:
```bash
kubectl apply -f blue-deployment.yaml
```

### Check the deployment status: 
```bash
kubectl get deploy
kubectl get pods -l env=blue
```

![blue-deployment](/blue-deployment.PNG)


### Exposing the Blue Deployment
The blue-green-service.yaml is responsible for routing traffic to our blue pods using a LoadBalancer Service. The traffic routing occurs because the selector values in blue-deployment.yaml (``spec:selector:matchLabels``) match the selector values defined in the blue-green-service.yaml (`spec:selector`).

This is how Kubernetes ensures that traffic is directed to the correct set of pods (in this case, the blue pods running version 1.0). Here's the service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-green-svc
spec:
  type: LoadBalancer
  selector:
    app: my-node-app
    env: blue
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
```
In this manifest:

- The selector in blue-green-service.yaml specifies that the service should route traffic to pods labeled app: my-node-app and env: blue. This matches the labels set in the blue-deployment.yaml manifest.

- As a result, all incoming traffic through this LoadBalancer Service will be routed to the blue pods running version 1.0 of the application.

### Deploy the service:
```bash
kubectl apply -f blue-green-service.yaml
```
### Check the service and confirm traffic routing:
```bash
kubectl get svc
curl http://<EXTERNAL_IP>
```

![curl-blue-service.PNG](/curl-blue-service.PNG)

## Setup the Green Environment

At this stage, we will deploy the new version of the application (v2.0) by defining a new deployment manifest green-deployment.yaml. The green environment will be deployed alongside the blue environment, but traffic will still be routed to the blue pods until we update the service to point to the green ones.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-deploy
  labels:
    app: my-node-app
    env: green
    version: v2.0
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-node-app
      env: green 
  template:
    metadata:
      labels:
        app: my-node-app
        env: green
        version: v2.0
    spec:
      containers:
      - name: my-node-app
        image: mik3asg/k8s-zero-downtime-deployment:v2.0
        ports:
        - containerPort: 3000
```

Here:

- The labels applied to the green pods (`app:my-node-app`, `env:green`, `version:v2.0`) differentiate them from the blue pods running the older version.
- The selector in this deployment (`app:my-node-app`, `env:green`) will manage these green pods, ensuring Kubernetes knows which pods belong to this new environment.


### Deploy the Green Environment
```bash
kubectl apply -f green-deployment.yaml
```
![blue-green-deploy](/blue-green-deploy.PNG)

At this point, both the blue (v1.0) and green (v2.0) pods are running in parallel, but traffic is still being routed to the blue environment.


## Update the Service to Re-route traffic from Blue to Green

To direct traffic to the green pods (version 2.0), we need to modify the existing service. This involves updating the selector in blue-green-service.yaml to match the labels of the green deployment. By doing so, we switch traffic from the blue environment to the green environment without any downtime.

Here's the updated service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-green-svc
spec:
  type: LoadBalancer
  selector:
    app: my-node-app
    env: green  # update from blue to green for migration
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
```
- The selector in the updated blue-green-service.yaml is now set to env: green, meaning the service will route traffic to the green pods (v2.0) instead of the blue ones.
- Since the selector values in the green-deployment.yaml (`spec:selector:matchLabels`) now match those in the service (`spec:selector`), all traffic will flow to the green pods.

### Apply the updated service configuration:
```bash
kubectl apply -f blue-green-service.yaml
```

### Check the status to confirm the traffic is routed to the green environment:

```bash
kubectl get svc
curl http://<EXTERNAL_IP>
```

![curl-blue-service.PNG](/curl-blue-service.PNG)

## Rolling Back to the Blue Environment
In a blue-green deployment, rolling back is straightforward because both environments (blue and green) are running simultaneously. If any issues are found in the green deployment (v2.0), you can quickly revert traffic back to the stable blue environment (v1.0) by updating the service selector.

To roll back traffic from the green pods (v2.0) to the blue pods (v1.0), we need to update the blue-green-service.yaml again. By changing the service’s selector back to env: blue, we ensure that the LoadBalancer routes traffic to the blue pods.

Here's how to update the service for rollback:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-green-svc
spec:
  type: LoadBalancer
  selector:
    app: my-node-app
    env: blue  # Rollback update: route traffic back to blue (v1.0)
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
```
In this manifest:

- The selector has been changed back to env: blue (as indicated in the comment), which matches the labels of the blue pods (v1.0). 
- This change re-routes all traffic back to the blue environment, restoring the previous stable version of the application.
To perform the rollback, apply the updated service configuration:
```bash
kubectl apply -f blue-green-service.yaml
```
### Verify that the service is now routing traffic to the blue pods:
```bash
kubectl get svc
curl http://<EXTERNAL_IP>
```

# Conclusion
Blue-green deployments in Kubernetes allow for zero-downtime updates by running both the old (blue) and new (green) versions of an application in parallel. This strategy ensures that traffic can be seamlessly switched between versions by simply updating the service’s selector. In case of issues with the new version, rolling back to the stable version is quick and risk-free.
## Why Rollback is Efficient in Blue-Green Deployments
Immediate Recovery: Since the blue environment is always running, traffic can be quickly redirected back without needing to redeploy the stable version.
Minimal Risk: Rolling back is as simple as updating the service selector, with no need to terminate the green pods or affect user traffic.
This approach provides flexibility, reduces risk, and ensures a smooth deployment process, making it ideal for production environments where uptime is critical.

