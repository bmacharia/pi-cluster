# Setting Up GitOps with Flux on a Kubernetes Cluster

This document provides a detailed guide to setting up a GitOps workflow using Flux on a freshly deployed Kubernetes cluster. A dedicated GitHub repository will be created for the project. Optional configurations, such as using a DevContainer, are covered in the bonus material.

---

## **Prerequisites**

- A functional Kubernetes cluster.
- A GitHub Personal Access Token (PAT) with the necessary permissions.

---

## **Creating a GitHub Personal Access Token**

1. Navigate to **GitHub Settings > Developer Settings > Personal Access Tokens**.
2. Generate a new classic token with the following permission:
   - `repo`: Full control of private repositories.
3. Save the token securely and set it as an environment variable:
   ```bash
   export GITHUB_TOKEN=<your-token>
   ```
4. Export your GitHub username as an environment variable:
   ```bash
   export GITHUB_USER=<your-username>
   ```

---

## **Installing the Flux CLI**

### **Installation Methods**

- Flux CLI can be installed using various methods, such as:
  - Homebrew:
    ```bash
    brew install fluxcd/tap/flux
    ```
  - Curl:
    ```bash
    curl -s https://fluxcd.io/install.sh | sudo bash
    ```

### **Verify Installation**

Ensure the Flux CLI is installed correctly:

```bash
which flux
```

---

## **Checking Cluster Compatibility**

Run the following command to validate that the cluster meets the prerequisites for installing Flux:

```bash
flux check --pre
```

---

## **Bootstrapping Flux on the Kubernetes Cluster**

Bootstrap Flux to integrate it with your GitHub repository. This process installs the necessary controllers and commits the required manifests to the specified repository:

### **Command**

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=pi-cluster \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

### **Explanation**

- `--owner`: Specifies the GitHub user or organization that owns the repository.
- `--repository`: Name of the repository to be bootstrapped.
- `--branch`: The Git branch to use for commits.
- `--path`: Path within the repository for cluster configuration manifests.
- `--personal`: Indicates the use of a personal GitHub account.

---

## **Results of the Bootstrap Process**

1. Flux creates the necessary Kubernetes manifests in the specified GitHub repository.
2. A new namespace, `flux-system`, is created on the Kubernetes cluster. This namespace contains all the Flux components.
3. The GitOps controller becomes active, continuously reconciling the desired state from the Git repository with the actual cluster state.

---
