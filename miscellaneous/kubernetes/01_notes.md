# Kubernetes Core Concepts and Pod Configuration

### Key Takeaways

- **Kubernetes** is a container orchestration platform that coordinates the collaboration of **Master Nodes** and **Worker Nodes**.
- **Master Nodes** (Control Plane) are responsible for scheduling and deciding where applications run.
- **Worker Nodes** provide the infrastructure to actually run the applications.
- In a **single-node cluster**, your computer plays the role of both Master and Worker Node.

### Containers

- Containers run applications in **isolation** with their **dependencies**, making them **highly portable**.
- Kubernetes does **not run containers directly**. It uses a **container runtime** like Docker, containerd, etc.

### Pods

- In Kubernetes, **pods** are the **smallest deployable units**.
- A **pod encapsulates one or more containers**, which share networking and storage.
- Kubernetes puts containers **inside pods** to allow it to automate scaling, self-healing, and resource allocation.

### Purpose

- A pod is used to run a single instance of a microservice.
- If the microservice needs additional functionality (like logging, proxying), it can use **sidecar containers**.

## Pod Configuration Details

### Metadata

- **`name`**: A unique name for the pod. Typically matches the microservice name due to their 1:1 relationship.
- **`labels`**: Key-value pairs that categorize the pod. Useful for grouping, querying, and service matching.

### Runtime Requirements (under `spec`)

- **`containers`**: The list of containers in the pod. Each container must specify:
  - `name`: Unique within the pod.
  - `image`: The Docker image to use.
  - `ports`: Container ports to expose.
  - `resources`: Resource management for memory and CPU.

#### Resources

- **`requests`**: The **minimum** resources needed. Kubernetes uses this to schedule the pod on a suitable node.
- **`limits`**: The **maximum** resources allowed.

##### Memory (Incompressible Resource)

- Memory **cannot be throttled**.
- If memory usage **exceeds the limit**, the container is **killed (OOMKilled)**.
- **Always define a memory limit.**

##### CPU (Compressible Resource)

- CPU **can be throttled**.
- If the container exceeds its CPU limit, the Linux kernel will **slow it down**, not kill it.
- **It's optional to set a CPU limit** — many recommend to avoid it to allow apps to burst when needed.

## Multi-Container Pods and Sidecars

- A pod can have multiple containers that share the same IP and storage.
- Useful for **sidecar patterns**:
  - Example: A proxy, logger, or config reloader running next to the main app.
- These containers are **created, scaled, and destroyed together**.

## Port Forwarding in Kubernetes

- Port forwarding connects a local port to a port on a pod inside the cluster.
- Used for **debugging**, **testing**, and **accessing internal-only services**.

### Command Structure

```bash
kubectl port-forward <pod-name> <local-port>:<pod-port>
```

### Example

```bash
kubectl port-forward grade-submission-portal 8080:5001
```

This forwards your local port 8080 to the pod's internal port 5001.

## Commands

```bash
# Go to project directory
cd kubernetes-practice/section-one/

# Deploy the pod defined in YAML
kubectl apply -f grade-submission-portal-pod.yaml

# Start minikube with Docker driver and deploy pod
minikube start --driver=docker && kubectl apply -f grade-submission-portal-pod.yaml

# Check running pods
kubectl get pods

# View detailed info about a pod
kubectl describe pod grade-submission-portal

# View logs of the container inside the pod
kubectl logs grade-submission-portal
kubectl logs -f grade-submission-portal             # Follow the logs live
kubectl logs -f grade-submission-portal -c grade-submission-portal  # If multiple containers

# Forward local port to pod port
kubectl port-forward grade-submission-portal 8080
kubectl port-forward grade-submission-portal 8080:5001

# Delete pod
kubectl delete pod grade-submission-portal
```

---

## YAML Example: Pod Definition for grade-submission-portal

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grade-submission-portal # Microservice name, used internally to refer to this Pod
  labels:
    app.kubernetes.io/name: grade-submission # Label that identifies the overall application the pod is part of
    app.kubernetes.io/component: frontend # Label specifying the role/component (frontend/backend/etc.) of the microservice
    app.kubernetes.io/instance: grade-submission-portal # Label that distinguishes between different instances of the same application
spec:
  containers:
    - name: grade-submission-portal
      image: rslim087/kubernetes-course-grade-submission-portal
      resources:
        requests:
          memory: '128Mi' # Minimum memory required
          cpu: '200m' # Minimum CPU required
        limits:
          memory: '128Mi' # Maximum memory allowed (no CPU limit to avoid throttling)
      ports:
        - containerPort: 5001 # Port on which the app is exposed inside the pod
```
