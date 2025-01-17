# Kubernetes Home Lab

## Overview

This repository serves as a comprehensive guide for building and managing a Kubernetes cluster in a home lab environment. Specifically, it documents the setup of a single-node k3s cluster running on a Raspberry Pi cluster. This project integrates GitOps principles using Flux and employs a monorepo repository structure to manage configurations and deployments. The goal is to provide a streamlined, hands-on experience with Kubernetes and container orchestration. In the near future, additional nodes will be added to expand the cluster.

## Features

- Single-node k3s cluster setup on Raspberry Pi hardware
- GitOps implementation using Flux for continuous delivery
- Monorepo structure for managing Kubernetes manifests and configurations
- Deployment configurations for essential services (e.g., networking, storage, monitoring)
- Examples of containerized application deployments
- Tutorials and documentation covering various Kubernetes concepts

## Architecture

The Kubernetes cluster in this project includes the following components:

- **Single Node (k3s):** A lightweight Kubernetes distribution that simplifies cluster management.
- **GitOps Workflow:** Automatically synchronizes cluster state with the Git repository using Flux.
- **Networking:** Implements pod-to-pod communication using k3s’ built-in CNI.
- **Storage:** Provides persistent storage for stateful workloads.

## Prerequisites

Ensure that the following requirements are met before setting up the Kubernetes home lab:

### Hardware

- Raspberry Pi devices (minimum 1 for the single-node setup)
- Each Raspberry Pi should have at least 2 GB of RAM and a quad-core processor
- SD card (32 GB or larger recommended)
- Reliable power supply and network connectivity

### Software

- **Operating System:** Raspberry Pi OS (64-bit preferred)
- **k3s:** Lightweight Kubernetes distribution
- **Flux CLI:** For managing GitOps workflows
- **Git:** Version control system for managing the monorepo

## Setup Instructions

### 1. Prepare Raspberry Pi

1. Flash the Raspberry Pi OS image onto the SD card.
2. Configure networking and enable SSH access.
3. Install necessary dependencies, such as `curl` and `git`.

### 2. Install k3s

Install k3s on the Raspberry Pi:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify the installation:

```bash
kubectl get nodes
```

### 3. Configure GitOps with Flux

1. Install the Flux CLI:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

2. Bootstrap Flux with your Git repository:

```bash
flux bootstrap github \
  --owner=<your-username> \
  --repository=kubernetes-home-lab \
  --branch=main \
  --path=./clusters/home-lab
```

3. Commit and push your Kubernetes manifests to the repository.

### 4. Verify Flux Installation

Ensure that Flux is synchronizing the cluster state with the repository:

```bash
kubectl get pods -n flux-system
```

## Example Deployments

### Deploying Linkding

1. Add the deployment configuration to the repository under `apps/base/linkding/`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090
```

2. Commit and push the changes to the repository.
3. Verify that Flux has deployed the application:

```bash
kubectl get pods
```

### Accessing Linkding

Use port forwarding to access the service:

```bash
kubectl port-forward svc/linkding 9090:9090
```

## Folder Structure

```
.
├── apps
│   ├── base
│   │   └── linkding
│   │       ├── deployment.yaml
│   │       ├── kustomization.yaml
│   │       └── namespace.yaml
│   ├── staging
│       └── linkding
│           └── kustomization.yaml
├── clusters
│   └── staging
│       ├── apps.yaml
│       └── flux-system
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
```

## Best Practices

- **Security:** Secure the cluster using firewalls, role-based access control (RBAC), and network policies.
- **Backup:** Regularly back up the SD card or cluster configuration to prevent data loss.
- **Updates:** Keep k3s and Flux up to date to ensure compatibility and security.

## Contributing

Contributions are encouraged! Please submit issues or pull requests to help enhance this project.

## License

This project is distributed under the MIT License. Refer to the LICENSE file for details.

---

Enjoy exploring Kubernetes with GitOps! 🚀
