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

The Service provides a stable endpoint for the application pods. The Ingress Controller handles incoming HTTP traffic and
