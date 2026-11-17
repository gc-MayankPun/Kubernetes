# Kubernetes Learning Project

## Table of Contents

1. [Basic Idea](#basic-idea)
2. [Why Kubernetes](#why-kubernetes)
3. [Kubernetes Architecture](#kubernetes-architecture)
   - [Control Plane](#control-plane)
   - [Worker Node](#worker-node)
4. [Core Kubernetes Concepts](#core-kubernetes-concepts)
   - [Pod](#1-pod)
   - [Deployment](#2-deployment)
   - [ReplicaSet](#3-replicaset)
   - [Service](#4-service)
   - [Ingress](#5-ingress)
   - [Ingress Controller](#6-ingress-controller)
5. [How Everything Connects](#how-everything-connects)
6. [Project Structure](#project-structure)
7. [Local Kubernetes Setup](#local-kubernetes-setup)
   - [Technology Stack](#technology-stack)
   - [Verify the Cluster](#verify-the-cluster)
8. [Building and Running the Application](#building-and-running-the-application)
   - [Build the Docker Image](#step-1-build-the-docker-image)
   - [Import the Image into containerd](#step-2-import-the-image-into-containerd)
   - [Apply Kubernetes Manifests](#step-3-apply-kubernetes-manifests)
   - [Verify the Deployment](#step-4-verify-the-deployment)
   - [Access the Application](#step-5-access-the-application)
9. [Understanding imagePullPolicy](#understanding-imagepullpolicy)
10. [Updating the Application](#updating-the-application)
11. [Labels and Selectors](#labels-and-selectors)
12. [Useful Debugging Commands](#useful-debugging-commands)

- [Cluster](#cluster)
- [Pods](#pods)
- [Deployments](#deployments)
- [Services and Ingress](#services-and-ingress)
- [Common Errors](#common-errors)

13. [Final Takeaway](#final-takeaway)

---

## Basic Idea

Codebase + Dependencies + Runtime + OS is what makes a server, and when we bring all these into one file, it is called an **Image**.

And when we run this Image, we get a **Container**.

Running the same Image again and again will get us multiple new containers.

## Why Kubernetes

Kubernetes is used to automate the deployment, scaling and management of containerized applications, enabling high availability and efficient resource utilization.

And not only this...

We don't deploy an application just once. We continuously make changes, fix bugs, add features and release new versions.

Normally, shutting down Version 1 before starting Version 2 could cause downtime.

This is where Kubernetes helps us!

Some of the things Kubernetes handles:

- **Deployment:** Deploys and manages application workloads.
- **Self-healing:** Replaces failed or unhealthy pods.
- **Scaling:** Runs more or fewer replicas depending on requirements.
- **Rolling updates:** Gradually replaces old pods with new ones.
- **Rollback:** Allows us to return to a previous deployment revision.
- **Networking:** Provides stable communication between application components.

## Kubernetes Architecture

A Kubernetes cluster consists of two main parts.

### Control Plane

The control plane manages the cluster and makes decisions about what should run and where.

| Component          | Responsibility                                               |
| ------------------ | ------------------------------------------------------------ |
| API Server         | Entry point for Kubernetes API requests and kubectl commands |
| Scheduler          | Decides which node should run a newly created pod            |
| Controller Manager | Runs controllers that maintain the desired cluster state     |
| etcd               | Stores the cluster's configuration and state                 |

### Worker Node

Worker nodes are where application pods actually run.

| Component         | Responsibility                                                                 |
| ----------------- | ------------------------------------------------------------------------------ |
| kubelet           | Communicates with the control plane and manages pods on the node               |
| Container Runtime | Runs containers. This project uses containerd                                  |
| kube-proxy        | Implements Service networking rules, depending on the cluster networking setup |

In this project, the control plane and workload pods run on the same physical machine. The control-plane taint was removed to allow application workloads on the single-node cluster.

## Core Kubernetes Concepts

### 1. Pod

A Pod is the smallest deployable unit in Kubernetes.

It wraps one or more containers. Containers inside the same pod share the pod's network namespace and can communicate through `localhost`.

In practice, most application pods contain one main application container.

Each pod gets its own IP address. This IP can change when the pod is recreated, which is why we generally communicate with pods through Services rather than hardcoding pod IPs.

Useful commands:

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -f
```

### 2. Deployment

A Deployment manages application pods.

We describe the desired state in a YAML file, including:

- Container image
- Number of replicas
- Labels and selectors
- Container ports
- CPU and memory requests and limits

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: express-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: express
  template:
    metadata:
      labels:
        app: express
    spec:
      containers:
        - name: express-container
          image: cohort_2_express:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
            requests:
              memory: "64Mi"
              cpu: "250m"
```

The `replicas: 2` field tells Kubernetes to maintain two application pods.

If one pod crashes or is deleted, Kubernetes works to create a replacement.

Useful commands:

```bash
kubectl get deployments
kubectl describe deployment express-deployment
kubectl scale deployment express-deployment --replicas=3
kubectl rollout status deployment express-deployment
kubectl rollout history deployment express-deployment
kubectl rollout undo deployment express-deployment
kubectl rollout restart deployment express-deployment
```

### 3. ReplicaSet

A ReplicaSet ensures that the desired number of pod replicas are running.

The relationship looks like this:

```text
Deployment
    |
    v
ReplicaSet
    |
    +-------- Pod 1
    |
    +-------- Pod 2
```

A Deployment creates and manages ReplicaSets. When we update the pod template, Kubernetes creates a new ReplicaSet and gradually replaces the old pods.

We generally use Deployments directly instead of creating ReplicaSets ourselves because Deployments also provide rolling updates and rollback history.

Check ReplicaSets:

```bash
kubectl get replicasets
kubectl describe replicaset <replicaset-name>
```

### 4. Service

Pods are temporary. Their IP addresses can change whenever they are recreated.

A Service provides a stable network endpoint and routes traffic to matching pods using labels and selectors.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: express-service
spec:
  selector:
    app: express
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Here:

| Field        | Meaning                                       |
| ------------ | --------------------------------------------- |
| `selector`   | Finds pods labelled `app: express`            |
| `port`       | Port exposed by the Service                   |
| `targetPort` | Port on which the Express application listens |
| `type`       | Controls how the Service is exposed           |

Three common Service types:

- **ClusterIP:** Accessible inside the cluster. Default Service type.
- **NodePort:** Exposes a Service through a port on each node.
- **LoadBalancer:** Requests an external load balancer, generally provided by a cloud platform or a separate load-balancer implementation.

### 5. Ingress

Ingress defines HTTP and HTTPS routing rules.

For example, it can route requests to different Services based on a hostname or URL path.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: express-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: express-service
                port:
                  number: 80
```

This configuration sends requests matching `/` to `express-service` on port 80.

An Ingress resource only defines the rules. It does not handle network traffic by itself.

### 6. Ingress Controller

An Ingress Controller is the component that watches Ingress resources and actually handles incoming traffic.

This project uses **NGINX Ingress Controller**.

The difference:

| Ingress                           | Ingress Controller                        |
| --------------------------------- | ----------------------------------------- |
| Defines routing rules             | Processes incoming traffic                |
| Kubernetes resource               | Running controller workload               |
| Created through YAML              | Installed into the cluster                |
| Specifies where traffic should go | Routes traffic according to Ingress rules |

## How Everything Connects

The application request follows this path:

```text
                 Browser
                    |
                    v
              localhost:80
                    |
                    v
             NGINX Ingress
                    |
                    v
             Ingress Resource
                    |
                    v
             express-service
                    |
             +------+------+
             |             |
             v             v
           Pod 1         Pod 2
             |             |
             +------+------+
                    |
                    v
              Express App
                 :3000
```

The Service provides a stable endpoint for the application pods. The Ingress Controller handles incoming HTTP traffic and handles it according to the routing rules defined in the Ingress resource.

The Service then forwards the request to one of the available application pods.

This allows us to expose our application without directly exposing individual pods.

---

## Project Structure

The project is organized into two main parts:

- Application code
- Kubernetes configuration files

```text
.
├── k8s/
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
│
├── Dockerfile
├── package.json
├── package-lock.json
├── server.js
└── README.md
```

Here:

| File / Directory | Responsibility |
|---|---|
| `k8s/` | Contains Kubernetes manifests |
| `deployment.yml` | Defines application pods and replicas |
| `service.yml` | Provides stable networking for application pods |
| `ingress.yml` | Defines HTTP routing rules |
| `Dockerfile` | Defines how the application image is built |
| `server.js` | Express application entry point |
| `package.json` | Contains application dependencies and scripts |

The Kubernetes manifests describe the desired state of our application.

Instead of manually creating pods, services and networking configurations, we define them in YAML files and let Kubernetes manage them.

## Local Kubernetes Setup

This project runs on a local Kubernetes environment using Arch Linux.

### Technology Stack

| Technology | Purpose |
|---|---|
| Arch Linux | Host operating system |
| Docker | Building application images |
| Kubernetes | Container orchestration |
| containerd | Container runtime |
| kubectl | Communicating with the Kubernetes API |
| NGINX Ingress Controller | Handling incoming HTTP traffic |
| Express.js | Backend application |

### Verify the Cluster

Before deploying anything, we should verify that our Kubernetes cluster is running correctly.

Check the current context:

```bash
kubectl config current-context
```

Check available nodes:

```bash
kubectl get nodes -o wide
```

Check all running pods:

```bash
kubectl get pods -A
```

Check the container runtime:

```bash
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.containerRuntimeVersion}'
```

A node should have the `Ready` status before we proceed.

## Building and Running the Application

Now that the cluster is ready, we can deploy our Express application.

### Step 1: Build the Docker Image

First, we build an image using the Dockerfile.

```bash
docker build -t cohort_2_express:latest .
```

Check whether the image was created:

```bash
docker images
```

An image is a packaged version of our application containing the application code, dependencies and runtime environment.

### Step 2: Import the Image into containerd

One important thing to understand is that Docker and containerd maintain their own image storage.

Building an image using Docker does not automatically make it available to Kubernetes when the cluster uses a separate containerd image store.

We need to make the image available to the container runtime used by our Kubernetes node.

Save the image:

```bash
docker save cohort_2_express:latest -o cohort_2_express.tar
```

Import it into containerd:

```bash
sudo ctr -n k8s.io images import cohort_2_express.tar
```

Verify that the image is available:

```bash
sudo ctr -n k8s.io images ls
```

The `k8s.io` namespace is important because Kubernetes uses this containerd namespace for its images.

**Remember:** An image available through `docker images` is not necessarily available to the Kubernetes container runtime.

### Step 3: Apply Kubernetes Manifests

Once the image is available, we can apply our Kubernetes configurations.

Apply the Deployment:

```bash
kubectl apply -f k8s/deployment.yml
```

Apply the Service:

```bash
kubectl apply -f k8s/service.yml
```

Apply the Ingress:

```bash
kubectl apply -f k8s/ingress.yml
```

Alternatively, apply the entire directory:

```bash
kubectl apply -f k8s/
```

Kubernetes reads these manifests and works towards the desired state defined in them.

### Step 4: Verify the Deployment

Check the Deployment:

```bash
kubectl get deployments
```

Check ReplicaSets:

```bash
kubectl get replicasets
```

Check Pods:

```bash
kubectl get pods -o wide
```

Check Services:

```bash
kubectl get services
```

Check Ingress:

```bash
kubectl get ingress
```

For more detailed information:

```bash
kubectl describe deployment express-deployment
kubectl describe service express-service
kubectl describe ingress express-ingress
```

We can also check whether the Service has successfully discovered our application pods:

```bash
kubectl get endpoints express-service
```

If the Service has matching healthy pod endpoints, their IP addresses and target ports should appear.

The modern alternative is:

```bash
kubectl get endpointslices
```

### Step 5: Access the Application

Our application is exposed through the NGINX Ingress Controller.

The request flow is:

```text
Browser
   |
   v
localhost:80
   |
   v
NGINX Ingress Controller
   |
   v
Ingress Resource
   |
   v
express-service
   |
   v
Application Pod
   |
   v
Express Application :3000
```

For local testing, we can access the application using:

```text
http://localhost/
```

This assumes that the Ingress Controller is reachable through localhost on port 80.

If the controller is not exposed on localhost, we need to use the appropriate node address or port forwarding configuration.

---

## Understanding imagePullPolicy

This is one of the important concepts when working with locally built images.

Kubernetes provides three image pull policies:

| Policy | Behaviour |
|---|---|
| `Always` | Resolves the image reference through the registry whenever a container starts |
| `IfNotPresent` | Uses the local image if it is already available; otherwise, pulls it |
| `Never` | Never attempts to pull the image |

Example:

```yaml
containers:
  - name: express-container
    image: cohort_2_express:latest
    imagePullPolicy: IfNotPresent
```

### Always

With `Always`, Kubernetes asks the container runtime to resolve the image reference whenever a container starts.

If the resolved image is already cached, its existing layers can be reused.

It does not necessarily mean that the entire image is downloaded every time.

### IfNotPresent

With `IfNotPresent`, Kubernetes checks whether the image is available locally.

If it is available, the local image is used.

Otherwise, Kubernetes attempts to pull it.

This is useful when working with locally built images.

### Never

With `Never`, Kubernetes only uses an image that is already available locally.

If the image is missing, the container cannot start.

### Important Observation

Changing `imagePullPolicy` in a Deployment does not automatically modify already running Pods.

The policy is part of the Pod template.

When we update the Pod template, Kubernetes creates a new ReplicaSet and gradually replaces the old Pods.

Also, changing `Always` to `IfNotPresent` does not necessarily mean that `Always` was incorrect.

An image can still run successfully with `Always` if the container runtime can resolve it and the image is available or can be pulled.

**The important thing is understanding where the image exists and which container runtime Kubernetes is using.**

---

## Updating the Application

Applications are not deployed just once.

We continuously make changes to the code, fix bugs and introduce new features.

When the application changes, we need to update the image and make Kubernetes run the updated version.

### Step 1: Make Changes to the Application

Modify the Express application code.

### Step 2: Build the Updated Image

```bash
docker build -t cohort_2_express:latest .
```

### Step 3: Import the Updated Image

Export the image:

```bash
docker save cohort_2_express:latest -o cohort_2_express.tar
```

Import it into containerd:

```bash
sudo ctr -n k8s.io images import cohort_2_express.tar
```

If the image reference already exists, ensure that the imported image replaces or updates the reference used by the Pod.

### Step 4: Restart the Deployment

```bash
kubectl rollout restart deployment express-deployment
```

This tells Kubernetes to recreate the Pods managed by the Deployment.

Check the rollout:

```bash
kubectl rollout status deployment express-deployment
```

Verify the new Pods:

```bash
kubectl get pods -o wide
```

### Important

Rebuilding an image with the same tag does not automatically tell Kubernetes that the application has changed.

For example:

```yaml
image: cohort_2_express:latest
```

The tag remains the same even if the actual image contents change.

That is why we need to trigger a rollout when using locally rebuilt images with the same tag.

In production, using unique image tags or immutable image digests is generally a better approach.

---

## Labels and Selectors

Labels are key-value pairs attached to Kubernetes objects.

They help us organize and identify resources.

Example:

```yaml
metadata:
  labels:
    app: express
```

A Selector is used to find Kubernetes resources based on their labels.

For example, our Service uses:

```yaml
spec:
  selector:
    app: express
```

This tells the Service to find Pods carrying the label:

```yaml
app: express
```

The relationship looks like this:

```text
                 Service
                    |
                    |
              Selector
             app: express
                    |
             +------+------+
             |             |
             v             v
           Pod 1         Pod 2
         app: express   app: express
```

If the labels and selectors do not match, the Service cannot discover the intended Pods.

This is one reason why checking endpoints is useful when debugging networking problems.

Useful commands:

```bash
kubectl get pods --show-labels
```

```bash
kubectl get pods -l app=express
```

```bash
kubectl describe service express-service
```

---

## Useful Debugging Commands

When something goes wrong, we should not randomly change configurations.

Instead, inspect the cluster and identify where the problem is happening.

### Cluster

```bash
kubectl cluster-info

kubectl get nodes -o wide

kubectl describe node <node-name>

kubectl get events --sort-by=.metadata.creationTimestamp
```

### Pods

```bash
kubectl get pods -o wide

kubectl describe pod <pod-name>

kubectl logs <pod-name>

kubectl logs <pod-name> -f

kubectl logs <pod-name> --previous

kubectl get pods --show-labels
```

### Deployments

```bash
kubectl get deployments

kubectl describe deployment express-deployment

kubectl rollout status deployment express-deployment

kubectl rollout history deployment express-deployment

kubectl rollout undo deployment express-deployment

kubectl rollout restart deployment express-deployment
```

### Services and Ingress

```bash
kubectl get services

kubectl describe service express-service

kubectl get endpoints express-service

kubectl get endpointslices

kubectl get ingress

kubectl describe ingress express-ingress
```

Check the NGINX Ingress Controller:

```bash
kubectl get pods -n ingress-nginx

kubectl get services -n ingress-nginx

kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### Common Errors

**1. ImagePullBackOff**

Kubernetes cannot successfully pull the requested image.

Possible reasons:

- Incorrect image name or tag.
- Image does not exist in the registry.
- Registry authentication issues.
- Incorrect image pull policy.
- Locally built image is not available to the container runtime.

Useful commands:

```bash
kubectl describe pod <pod-name>

sudo ctr -n k8s.io images ls
```

**2. CrashLoopBackOff**

The container repeatedly starts and crashes.

Possible reasons:

- Application errors.
- Incorrect environment variables.
- Missing dependencies.
- Incorrect application startup command.
- Application listening on an unexpected port.

Useful commands:

```bash
kubectl logs <pod-name>

kubectl logs <pod-name> --previous

kubectl describe pod <pod-name>
```

**3. Service Has No Endpoints**

The Service cannot find matching Pods.

Possible reasons:

- Incorrect Service selector.
- Incorrect Pod labels.
- Pods are not ready.
- Application Pods are not running successfully.

Useful commands:

```bash
kubectl get pods --show-labels

kubectl describe service express-service

kubectl get endpoints express-service

kubectl get endpointslices
```

**4. Ingress Not Working**

Possible reasons:

- NGINX Ingress Controller is not running.
- Incorrect Ingress class.
- Incorrect Service name or port.
- Service has no available endpoints.
- Controller is not reachable through the expected address or port.

Useful commands:

```bash
kubectl get ingress

kubectl describe ingress express-ingress

kubectl get pods -n ingress-nginx

kubectl get services -n ingress-nginx
```

---

## Final Takeaway

Working with Kubernetes helped me understand that deploying an application is much more than simply running a container.

We need to understand how different components communicate and how Kubernetes maintains the desired state of our application.

The overall relationship looks like this:

```text
              Kubernetes Cluster
                      |
              +-------+-------+
              |               |
              v               v
         Control Plane     Worker Node
                              |
                              v
                           Kubelet
                              |
                              v
                            Pod
                              |
                              v
                          Container
                              |
                              v
                        Express App
```

And when a request comes from outside the cluster:

```text
Browser
   |
   v
Ingress Controller
   |
   v
Ingress Resource
   |
   v
Service
   |
   v
Pod
   |
   v
Container
```

Each component has a specific responsibility:

- **Deployment:** Maintains the desired number of application replicas.
- **ReplicaSet:** Ensures that the required number of Pods are running.
- **Pod:** Provides the environment in which our containers run.
- **Service:** Provides stable networking for Pods.
- **Ingress:** Defines external HTTP and HTTPS routing rules.
- **Ingress Controller:** Processes incoming traffic according to those rules.

Kubernetes brings all these components together to automate application deployment, scaling, networking and recovery.

**The biggest thing I learned:** Kubernetes is not just about running containers. It is about continuously maintaining the desired state of an entire application.