# Monolithic vs Microservices & Introduction to Kubernetes

## Monolithic Applications

- Applications are built as a single, layered structure.
- Difficult to maintain, update, or scale.
- A small change often requires redeploying the entire application.


## Microservices Architecture

- The application is split into many independent microservices.
- Each microservice performs a specific task.
- Each microservice runs inside a container.

### Container

- A lightweight, isolated environment.
- Includes only the resources and dependencies the app needs.
- Can run anywhere: local machine, data center, or cloud.

---

## Challenges with Microservices

1. **Resource Allocation**
   - How to efficiently distribute CPU and memory between containers?

2. **Resiliency** 
   - How to ensure the system recovers automatically from failure?

3. **Load Balancing** 
   - If an app is running out of memory or CPU, how do you dynamically scale it by adding instances?

---

## Kubernetes Overview

- Kubernetes is a container orchestration platform.
- It is software that runs across multiple physical or virtual machines.

### Kubernetes Cluster

- A group of connected machines.
- These machines are called **nodes** in Kubernetes terminology.

### Master Node

- The control plane or "brain" of the cluster.
- Responsible for making decisions:
  - When to deploy a container
  - Where to deploy it
  - How to monitor and manage it

### Worker Node

- Contains the infrastructure needed to run the actual container workloads.
- The containers are executed on these nodes.

---

## Pod: The Smallest Deployment Unit

- You cannot deploy a container directly in Kubernetes.
- Each container must be wrapped in a **Pod**.
- A pod is the **smallest deployable unit** in Kubernetes.

