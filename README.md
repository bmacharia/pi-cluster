# Kubernetes GitOps Platform

`pi-cluster` is a production-patterned Kubernetes platform built on k3s and operated through a GitOps workflow with Flux.

The repository acts as the declarative source of truth for application workloads, platform services, observability components, and cluster configuration. Flux continuously retrieves the desired configuration from Git, renders the appropriate Kustomize configuration, applies it to Kubernetes, and reconciles configuration drift when the live cluster diverges from the declared state.

The repository follows a layered platform structure that separates application configuration from platform infrastructure and operational services. Application manifests are organized using reusable Kustomize bases and environment-specific composition. Cluster-level Flux configuration defines which sections of the repository are independently reconciled, allowing applications, infrastructure controllers, infrastructure configuration, monitoring components, and test workloads to operate as separate reconciliation units.

## Delivery Model

The delivery path follows a controller-based model:

```text
Git repository
      ↓
Flux source-controller retrieves a Git revision
      ↓
Flux Kustomizations define reconciliation boundaries
      ↓
kustomize-controller renders and applies desired resources
      ↓
Kubernetes controllers converge workloads toward the resulting specifications
      ↓
Runtime and application-level validation confirms that the deployed system behaves as intended
```

This creates two complementary control loops.

Flux reconciles Git-defined desired state with Kubernetes objects, while native Kubernetes controllers reconcile those objects with running resources such as ReplicaSets, Pods, Services, and endpoints.

## Platform Responsibilities

The platform separates several operational domains:

- **Application delivery:** Kubernetes workloads are defined declaratively and composed through Kustomize bases and overlays.
- **GitOps control plane:** Flux manages source retrieval, reconciliation, pruning, dependency ordering, and drift correction.
- **Platform infrastructure:** Infrastructure controllers and their configuration are managed independently from application workloads, allowing controller dependencies and cluster services to be deployed in an ordered manner.
- **Observability:** Prometheus, Grafana, Alertmanager, and supporting monitoring components provide cluster and workload visibility.
- **Secrets management:** SOPS with age is used to support encrypted secrets stored alongside declarative configuration and decrypted by the GitOps workflow when applied.
- **Dependency automation:** Renovate configuration supports automated dependency and container-image update workflows.
- **Operations:** Runbooks, experiments, incident notes, and engineering documentation capture operational knowledge alongside the platform configuration.

## Git as the Source of Truth

The platform is designed around Git as the durable configuration authority rather than treating the Kubernetes API as the primary configuration store.

Manual cluster changes can temporarily alter runtime state, but Flux reconciliation restores Git-defined configuration unless the desired state is intentionally changed or reconciliation is suspended.

The reconciliation model can be summarized as:

```text
Git desired state
      ↓
Flux reconciliation
      ↓
Kubernetes objects
      ↓
Kubernetes controllers
      ↓
Running workloads
```

This design supports several production engineering practices:

- Repeatable deployments
- Auditable configuration changes
- Drift detection and correction
- Separation of concerns
- Declarative environment configuration
- Controlled platform dependencies
- Observability
- Encrypted secret management
- Evidence-based operational validation

## Engineering Purpose

Although the environment is a three-node homelab rather than a production cluster, its purpose is to reproduce the engineering patterns used to operate larger Kubernetes platforms.

It provides a controlled environment for practicing:

- GitOps operations
- Kubernetes reconciliation
- Failure investigation
- Configuration management
- Observability
- Recovery procedures
- Infrastructure automation
- Platform engineering
- Production-style troubleshooting and validation

The environment is intentionally used as an engineering laboratory for understanding how declarative systems behave, how control loops interact, how failures propagate, and how infrastructure changes can be validated with evidence—without claiming production-equivalent scale or failure consequences.