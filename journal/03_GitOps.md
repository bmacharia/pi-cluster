# GitOps Documentation

## **The Point of GitOps**

Is to have the ability yo delete a Kubernetes Cluster and point it to another piece of hardware and rollout with as few manual steps involved. I want to have as much possible automated

## **Definition of GitOps**

GitOps is a set of best practices that streamline the delivery of applications and infrastructure by leveraging Git as the single source of truth. This methodology ensures that:

- The entire code delivery process is managed via Git.
- Both infrastructure and application definitions are maintained as code.
- Updates and rollbacks are handled through automation.

---

## **Key Principles of GitOps**

1. **Declarative System Description**: The entire system, including infrastructure and applications, is described declaratively.
2. **Version-Controlled State**: The desired state of the system is stored and versioned in a Git repository.
3. **Automated Change Application**: Approved changes in Git are automatically applied to the system.
4. **Continuous Correctness Enforcement**: Software agents continuously monitor for divergence between the declared state in Git and the actual system state, providing alerts and reconciling as needed.

---

## **GitOps Workflow**

1. **Centralized Git Repository**: A single Git repository stores all configuration for applications and infrastructure.
2. **Cluster Sync Mechanism**: The cluster periodically pulls changes and deployment details from the Git repository.
3. **Reconciliation Loop**: A GitOps controller operates a continuous reconciliation loop to ensure that the actual cluster state matches the desired state as defined in Git.

---

## **Benefits of GitOps**

- **Version Control and History**: Provides a comprehensive version history of all changes.
- **Enhanced Collaboration**: Leverages Git workflows to improve team collaboration and code review processes.
- **Improved Security**: Reduces the need for direct cluster access by using automated processes.
- **Automation Advantages**: Simplifies updates and rollbacks, enhancing operational efficiency.

---

## **Flux vs. Argo CD**

For this course, Flux has been chosen due to the following advantages:

- **Simplified Setup**: Offers a less complex and more approachable interface.
- **CLI-Centric Workflow**: Encourages command-line interface (CLI) usage and reinforces GitOps principles.
- **AKS Integration**: Provides native integration with Azure Kubernetes Service (AKS).
- **Manifest Management**: Relies heavily on Kustomize for managing Kubernetes manifests.

---

## **Kustomize**

Kustomize is a powerful templating language for managing Kubernetes manifests. It enables:

- Declarative configuration management for Kubernetes objects.
- The creation of reusable and composable manifests.
- Streamlined customization of Kubernetes deployments.

As an essential tool for Kubernetes engineers, proficiency in Kustomize is crucial for modern Kubernetes workflows.

---

## **Using GitOps in a Home Lab**

Implementing GitOps in a home lab environment provides the following benefits:

- **Hands-On Experience**: Offers practical exposure to industry-standard practices.
- **Skill Demonstration**: Creates a public and verifiable record of learning and skills.
- **Collaboration and Control**: Facilitates collaboration and version control, even in personal projects.

---
