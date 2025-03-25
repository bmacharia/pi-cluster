# Kube-Prometheus-Stack Deployment with Flux

This documentation explains how to deploy the **kube-prometheus-stack** Helm chart using Flux. It will walk you through the structure of the `HelmRelease` and `Kustomization` files, their purpose, and how to access the Grafana dashboard once everything is installed. This guide is intended for a more technical audience looking to automate and manage their Kubernetes monitoring stack.

---

## Overview

- **kube-prometheus-stack** is a collection of Kubernetes manifests, Grafana dashboards, and Prometheus rules, providing a comprehensive monitoring solution.
- **Flux** is a GitOps tool that continuously monitors your Git repository for changes and applies them to your Kubernetes cluster.
- We use a **HelmRelease** object in Flux to deploy Helm charts automatically.
- A **Kustomization** resource is used to stitch together YAML manifests and apply them to your cluster in a structured manner.

---

## Prerequisites

1. A Kubernetes cluster up and running.
2. Flux installed and configured in your cluster.
3. A Git repository where your Kubernetes manifests are stored.
4. Basic understanding of Helm and Kustomize.

---

## The HelmRelease Definition

Below is the `HelmRelease` object which deploys the `kube-prometheus-stack` chart.

### File: `release.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "66.2.2"
      # Alternatively:
      # version: "58.x"
      sourceRef:
        kind: HelmRepository
        name: kube-prometheus-stack
        namespace: monitoring
      interval: 12h

  # CRD Installation Instructions
  install:
    crds: Create
  upgrade:
    crds: CreateReplace

  # Drift Detection
  driftDetection:
    mode: enabled
    ignore:
      # Ignore "validated" annotation which is not inserted during install
      - paths: ["/metadata/annotations/prometheus-operator-validated"]
        target:
          kind: PrometheusRule

  # Values used by the Chart
  values:
    grafana:
      adminPassword: mischa
```

## File: kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:

- namespace.yaml
- repository.yaml
- release.yaml

## Explanation of key fields

`apiVersion` and `kind`: Uses kustomize.config.k8s.io/v1beta1 and Kustomization for a native Kubernetes resource that manages the composition of Kubernetes manifests.

`resources:`

`namespace.yaml`: Contains the definition of the monitoring namespace.

`repository.yaml`: Contains the definition of the HelmRepository object used by Flux to pull the kube-prometheus-stack chart.

`release.yaml`: The HelmRelease manifest that you saw earlier.

## Applying the Manifests

Ensure your Flux instance is configured to watch your Git repository.

Commit and push your kustomization.yaml, release.yaml, namespace.yaml, and repository.yaml to the Git repository that Flux is monitoring.

Once Flux notices the changes (which is typically within a few seconds to a couple of minutes depending on your Flux settings), it will apply the Kustomization resource, which in turn applies:

## Verifying the Deployment

1. **Check the HelmRelease Status:**

```bash
kubectl get helmreleases -n monitoring
```

2. ** Check the Pods**

```bash
kubectl get pods -n monitoring
```

## Accessing Grafana

Once the `kube-prometheus-stack` is running, you can access **Grafana** via port-forwarding

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 8080:80
```

After running this commanf open your browser to `http://localhost:8080

Log in with

- Username: **admin**
- Password: **my-password**
