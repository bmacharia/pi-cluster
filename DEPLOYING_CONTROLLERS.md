# Deploying Infrastructure Controllers on this Cluster

This guide covers deploying **Helm-based infrastructure controllers** — things like load balancers, storage systems, and ingress controllers — in a GitOps way using Flux. This is different from deploying applications (see `DEPLOYING_APPS.md`). Controllers install CRDs and cluster-wide components. They require a different structure.

---

## The Key Difference: Controllers vs Apps

| Apps (`apps/`) | Controllers (`infrastructure/`) |
|---|---|
| Plain Deployments | Helm charts via HelmRelease |
| App-specific namespaces | Dedicated system namespaces |
| No CRDs | Install CRDs into the cluster |
| Single Kustomization | Two Kustomizations (controllers + configs) |

Controllers like MetalLB install their own Kubernetes resource types (CRDs). Once installed, you configure them using those custom resources. This creates a dependency: the configuration cannot be applied until the controller is running. This is why you need two separate Flux Kustomizations with a `dependsOn` relationship.

---

## Repository Structure

```
infrastructure/
├── controllers/
│   ├── base/
│   │   ├── kustomization.yaml          ← aggregate: lists all controller subdirs
│   │   └── <controller-name>/
│   │       ├── namespace.yaml          ← dedicated namespace
│   │       ├── repository.yaml         ← HelmRepository (chart source)
│   │       ├── release.yaml            ← HelmRelease (installs the controller)
│   │       └── kustomization.yaml      ← base inventory
│   └── staging/
│       ├── kustomization.yaml          ← aggregate: lists all controller subdirs
│       └── <controller-name>/
│           └── kustomization.yaml      ← references base, no CRD configs here
└── configs/
    └── staging/
        ├── kustomization.yaml          ← aggregate: lists all config subdirs
        └── <controller-name>/
            ├── kustomization.yaml      ← lists CRD config files
            ├── resource-a.yaml         ← CRD-based config (e.g. IPAddressPool)
            └── resource-b.yaml         ← CRD-based config (e.g. L2Advertisement)

clusters/staging/infrastructure.yaml    ← two Flux Kustomizations with dependsOn
```

---

## The Two-Kustomization Pattern

The Flux Kustomization CRDs live in `clusters/staging/infrastructure.yaml`. There are always two:

```yaml
# First: installs the controller and its CRDs via HelmRelease
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastucture-controllers
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./infrastructure/controllers/staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system

---
# Second: applies CRD-based configuration AFTER the controller is ready
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-configs
  namespace: flux-system
spec:
  interval: 1m0s
  dependsOn:
    - name: infrastucture-controllers   ← waits for controllers to be healthy first
  path: ./infrastructure/configs/staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

`dependsOn` is the critical field. Without it, Flux applies both Kustomizations simultaneously. The config Kustomization will fail because the CRDs do not exist yet. With `dependsOn`, Flux waits until `infrastucture-controllers` is `Ready: True` before attempting `infrastructure-configs`.

---

## Step-by-Step: Deploying a New Controller

Replace `<controller-name>` and `<namespace>` throughout.

---

### Step 1 — Namespace

**File:** `infrastructure/controllers/base/<controller-name>/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
```

---

### Step 2 — HelmRepository

**File:** `infrastructure/controllers/base/<controller-name>/repository.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: <controller-name>
  namespace: <namespace>
spec:
  interval: 24h
  url: https://<chart-repository-url>
```

This is the equivalent of `helm repo add`. Flux polls it every 24 hours for new chart versions. The `name` here is what the HelmRelease references in `sourceRef`.

---

### Step 3 — HelmRelease

**File:** `infrastructure/controllers/base/<controller-name>/release.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <controller-name>
  namespace: <namespace>
spec:
  interval: 30m
  chart:
    spec:
      chart: <chart-name>
      version: "<version>"
      sourceRef:
        kind: HelmRepository
        name: <controller-name>
        namespace: <namespace>
      interval: 12h
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  driftDetection:
    mode: enabled
  values: {}    # remove if no custom values needed
```

Key fields:
- `install.crds: Create` — installs the controller's CRDs on first deploy
- `upgrade.crds: CreateReplace` — updates CRDs when the chart version changes. Without this, CRD schema changes in new chart versions are silently ignored
- `driftDetection.mode: enabled` — Flux will revert manual changes to Helm-managed resources
- `version` — always pin to a specific version. Renovate bot will open PRs to bump it

---

### Step 4 — Base Kustomization

**File:** `infrastructure/controllers/base/<controller-name>/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - repository.yaml
  - release.yaml
```

No `namespace:` field. Base is environment-agnostic.

---

### Step 5 — Controllers Staging Overlay

**File:** `infrastructure/controllers/staging/<controller-name>/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <namespace>
resources:
  - ../../base/<controller-name>/
```

No CRD config files here — those go in `configs/`. The controllers overlay only pulls in the base.

---

### Step 6 — CRD Config Files

**Directory:** `infrastructure/configs/staging/<controller-name>/`

Create one YAML file per CRD-based configuration object. These are the custom resources the controller exposes after installation.

Example structure for MetalLB:
```
infrastructure/configs/staging/metallb/
  ip-address-pool.yaml    ← IPAddressPool CRD
  l2-advertisement.yaml   ← L2Advertisement CRD
```

Each file uses the controller's CRD `apiVersion` and `kind`. Find these in the controller's documentation.

---

### Step 7 — Configs Kustomization

**File:** `infrastructure/configs/staging/<controller-name>/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <namespace>
resources:
  - resource-a.yaml
  - resource-b.yaml
```

---

### Step 8 — Register in Aggregate Kustomizations

Add `<controller-name>` to both aggregate files:

**`infrastructure/controllers/base/kustomization.yaml`**
```yaml
resources:
  - renovate
  - <controller-name>
```

**`infrastructure/controllers/staging/kustomization.yaml`**
```yaml
resources:
  - renovate
  - <controller-name>
```

**`infrastructure/configs/staging/kustomization.yaml`**
```yaml
resources:
  - <controller-name>
```

---

## Deploying

```bash
git add infrastructure/ clusters/staging/infrastructure.yaml
git commit -m "feat: add <controller-name>"
git push

flux reconcile source git flux-system && flux reconcile kustomization infrastucture-controllers
```

Watch the HelmRelease install:

```bash
kubectl get helmrelease -n <namespace> -w
kubectl get pods -n <namespace> -w
```

Once controllers are `Ready`, Flux automatically reconciles `infrastructure-configs` and applies the CRD-based configuration.

---

## Verifying

```bash
# All Flux Kustomizations healthy
flux get kustomizations

# HelmRelease status
kubectl get helmrelease -n <namespace>

# Controller pods
kubectl get pods -n <namespace>

# CRD-based resources applied
kubectl get <crd-kind> -n <namespace>
```

---

## Troubleshooting

### CRD resources failing with "no matches for kind"

The controller's HelmRelease has not finished installing yet. The CRDs do not exist. Check the HelmRelease status:

```bash
kubectl describe helmrelease <controller-name> -n <namespace>
```

If `infrastructure-configs` does not have `dependsOn: infrastucture-controllers` set in `clusters/staging/infrastructure.yaml`, Flux applies both Kustomizations simultaneously and the configs will always fail on first apply.

### HelmRelease stuck in "reconciling"

```bash
kubectl describe helmrelease <controller-name> -n <namespace> | grep -A 10 "Conditions:"
```

Common causes:
- Chart version does not exist — verify the version against the Helm repository
- `sourceRef` namespace does not match where the HelmRepository was created
- Network issue pulling the chart — check `kubectl get helmrepository -n <namespace>`

### Flux kustomization build failed

```bash
kubectl describe kustomization infrastucture-controllers -n flux-system | grep "Message:"
```

Usually a YAML error in one of the files. The error message names the file and line number.

---

## File Checklist

```
infrastructure/controllers/base/<controller-name>/
  ✓ namespace.yaml
  ✓ repository.yaml       correct chart URL, name matches release sourceRef
  ✓ release.yaml          version pinned, crds: Create/CreateReplace set
  ✓ kustomization.yaml    lists all three files, no namespace field

infrastructure/controllers/staging/<controller-name>/
  ✓ kustomization.yaml    namespace set, references base only — no CRD configs

infrastructure/configs/staging/<controller-name>/
  ✓ kustomization.yaml    namespace set, lists all CRD config files
  ✓ <crd-config>.yaml     correct apiVersion (exact, no truncation)

infrastructure/controllers/base/kustomization.yaml
  ✓ <controller-name> added

infrastructure/controllers/staging/kustomization.yaml
  ✓ <controller-name> added

infrastructure/configs/staging/kustomization.yaml
  ✓ <controller-name> added

clusters/staging/infrastructure.yaml
  ✓ infrastructure-configs Kustomization exists with dependsOn: infrastucture-controllers
```
