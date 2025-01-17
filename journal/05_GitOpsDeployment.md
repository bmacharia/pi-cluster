# Deploying Linkding Using Flux and GitOps Principles

This guide will walk you through deploying the application, **Linkding** (a bookmark manager), to a Kubernetes cluster using **Flux** and **GitOps principles**. By following Flux's recommended repository structure and best practices, you'll achieve version-controlled, declarative management of your Kubernetes resources.

---

## Repository Structure

The repository is organized into separate directories for apps, base configurations, and environment-specific configurations (e.g., staging, production):

```
repository/
  ├── clusters/
  │     ├── staging/
  │     │     └── apps.yaml
  ├── apps/
  │     ├── staging/
  │     │     ├── kustomization.yaml
  │     │     ├── deployment.yaml
  │     │     └── namespace.yaml
```

---

## YAML File Descriptions

### 1. `apps.yaml`

This file tells Flux to monitor the `apps/staging` directory.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/staging/
```

### 2. `kustomization.yaml`

Defines resources and configurations for the application.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - namespace.yaml
```

### 3. `namespace.yaml`

Creates a dedicated namespace for the application.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: linkding
```

### 4. `deployment.yaml`

Defines the Kubernetes deployment for Linkding.

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

---

## Deployment Flow

Here's how Flux deploys the application:

1. Flux reads the `clusters/staging` directory.
2. Flux finds and applies the `apps.yaml` file.
3. It references the `apps/staging` directory specified in `apps.yaml`.
4. Flux identifies the `kustomization.yaml` file for Linkding.
5. The resources defined in the base configuration (e.g., `namespace.yaml`, `deployment.yaml`) are applied.

Once the configuration is committed and pushed, **Flux automatically detects and applies the new configuration** without any manual intervention.

---

## Verifying the Deployment

After deployment, verify the resources:

### 1. Check the Namespace

Run the following command to confirm the namespace creation:

```bash
kubectl get namespaces
```

### 2. Check the Pods

Ensure the Linkding pod is running:

```bash
kubectl get pods -n linkding
```

### 3. Access the Application

Use port-forwarding to access the Linkding application:

```bash
kubectl port-forward pod/linkding 8080:9090 -n linkding
```

Open your browser and navigate to `http://localhost:8080` to access Linkding.

---

## Benefits of GitOps with Flux

- **Declarative Management**: Kubernetes resources are defined in YAML files.
- **Version Control**: Configuration changes are tracked in Git.
- **Automation**: Flux continuously monitors and applies updates, reducing manual interventions.

This approach simplifies application deployment and management while ensuring reliability and consistency in your Kubernetes environment.
