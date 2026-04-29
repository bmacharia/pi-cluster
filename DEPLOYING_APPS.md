# Deploying Applications on this Cluster

This is a Flux-based GitOps cluster. You do not `kubectl apply` to deploy apps. You write YAML, push to Git, and Flux applies it. This document is the complete manual for deploying a new application.

---

## How Flux Works

Flux runs inside the cluster and watches this Git repository on a 1-minute interval. When it detects a change it builds the full Kubernetes resource graph using Kustomize and applies it to the cluster via server-side apply.

```
Git push → Flux polls GitHub (every 1m)
  → Kustomize builds resource graph from ./apps/staging
    → API server applies objects
      → kubelet schedules and starts Pods
```

The entry point is `clusters/staging/apps.yaml`. It points Flux at `./apps/staging`. Everything flows from there.

---

## Repository Structure

```
apps/
├── base/
│   └── <app-name>/          ← reusable blueprint, environment-agnostic
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── storage.yaml      ← only if the app needs persistent storage
│       └── kustomization.yaml
└── staging/
    ├── kustomization.yaml    ← aggregate: lists every app deployed to staging
    └── <app-name>/           ← staging-specific decisions (namespace, ingress)
        ├── kustomization.yaml
        └── ingress.yaml
```

**Base** is the blueprint. It knows nothing about which environment it runs in. No namespace field. No hostnames. No secrets.

**Staging overlay** is where environment-specific decisions live: which namespace to deploy into, what hostname the Ingress listens on, any environment-specific secrets.

**The aggregate kustomization** (`apps/staging/kustomization.yaml`) is the list of apps Flux knows about. Adding an app here is what makes Flux deploy it. Removing it is what makes Flux delete it (`prune: true`).

---

## Step-by-Step: Deploying a New App

Replace `<app-name>` with your application name throughout. Use lowercase-hyphenated names (e.g. `uptime-kuma`, `linkding`, `mealie`).

### Step 1 — Namespace

**File:** `apps/base/<app-name>/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <app-name>
```

Every app gets its own namespace. This is an isolation boundary — it scopes your `kubectl` commands, contains blast radius, and allows clean teardown (delete the namespace, everything inside goes with it).

---

### Step 2 — Storage (skip if the app has no state)

**File:** `apps/base/<app-name>/storage.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <app-name>-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Use this for any app that writes data to disk (databases, config files, media). Container filesystems are ephemeral — a Pod restart wipes them. A PVC survives Pod restarts, rolling updates, and node drains.

`ReadWriteOnce` means one node mounts it at a time. Correct for single-replica apps. No `storageClassName` needed — the cluster default (`local-path`) provisions it automatically.

Name convention: `<app-name>-data-pvc`. If an app has multiple data directories, name them descriptively: `<app-name>-config-pvc`, `<app-name>-media-pvc`.

---

### Step 3 — Deployment

**File:** `apps/base/<app-name>/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
        - name: <app-name>
          image: <image>:<tag>
          ports:
            - containerPort: <port>
              protocol: TCP
          volumeMounts:
            - mountPath: /path/to/data
              name: <app-name>-data
      volumes:
        - name: <app-name>-data
          persistentVolumeClaim:
            claimName: <app-name>-data-pvc
```

**Critical wiring — three names must form an unbroken chain:**

```
volumeMounts.name  →  volumes.name  →  volumes.persistentVolumeClaim.claimName
```

If any link in that chain is broken, the Pod will fail to schedule.

**On security contexts:**

Not all images need them. Only add `runAsUser` / `fsGroup` if you know the image is designed to run as a specific non-root UID from process start (check the image documentation). If the image entrypoint runs as root to set up permissions and then drops privileges itself (common pattern), adding `runAsUser` will break it.

```yaml
# Only add this block if the image explicitly supports it
securityContext:
  fsGroup: 1000
  runAsUser: 1000
```

**On image tags:**

Never use `latest`. Pin to a specific version tag or major tag (e.g. `1`, `2.8.0`). Renovate bot is configured in this cluster and will open PRs to bump image versions automatically.

---

### Step 4 — Service

**File:** `apps/base/<app-name>/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
spec:
  selector:
    app: <app-name>
  ports:
    - port: <port>
      targetPort: <port>
  type: ClusterIP
```

The Service gives your app a stable internal DNS name: `<app-name>.<namespace>.svc.cluster.local`. Pod IPs are ephemeral — the Service IP is not. The Ingress points at the Service, not the Pod.

The `selector` must match the `labels` on your Deployment's Pod template exactly. If it doesn't, the Service has zero endpoints and all traffic is silently dropped.

`ClusterIP` means internal only. External access is handled by the Ingress.

---

### Step 5 — Base Kustomization

**File:** `apps/base/<app-name>/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - storage.yaml
```

This is the inventory for the base. Kustomize does not glob directories — every file must be listed here. No `namespace:` field in the base kustomization. The base is environment-agnostic.

Remove `storage.yaml` from the list if you did not create it.

---

### Step 6 — Staging Overlay Kustomization

**File:** `apps/staging/<app-name>/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <app-name>
resources:
  - ../../base/<app-name>/
  - ingress.yaml
```

Two jobs:
1. `namespace: <app-name>` — Kustomize injects this into `metadata.namespace` on every resource built from the base. Without this, resources land in the wrong namespace.
2. `resources:` — pulls in the entire base, then layers on staging-specific files.

---

### Step 7 — Ingress

**File:** `apps/staging/<app-name>/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  ingressClassName: traefik
  rules:
    - host: <subdomain>.airmanbabu.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>
                port:
                  number: <port>
```

Ingress lives in the staging overlay because the hostname is environment-specific. Traefik watches for Ingress objects and programs its routing table. The `service.name` and `service.port.number` must match your Service exactly.

DNS: create an A or CNAME record for `<subdomain>.airmanbabu.net` pointing at the cluster's external IP before testing.

---

### Step 8 — Register with Flux

**File:** `apps/staging/kustomization.yaml`

Add your app to the resources list:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - linkding
  - audiobookshelf
  - mealie
  - uptime-kuma
  - <app-name>      ← add this line
```

This is the trigger. Until this line exists, Flux ignores every file you created. Once it is committed and pushed, Flux will deploy your app on the next reconcile cycle.

---

## Deploying

Commit all new files and push:

```bash
git add apps/base/<app-name>/ apps/staging/<app-name>/ apps/staging/kustomization.yaml
git commit -m "feat: add <app-name> deployment"
git push
```

Force immediate reconciliation (instead of waiting up to 1 minute):

```bash
flux reconcile source git flux-system && flux reconcile kustomization apps
```

Watch the Pod come up:

```bash
kubectl get pods -n <app-name> -w
```

---

## Verifying a Deployment

```bash
# All resources in the app namespace
kubectl get pods,svc,ingress,pvc -n <app-name>

# Pod logs
kubectl logs -n <app-name> deployment/<app-name>

# Previous Pod logs (if it is crash-looping)
kubectl logs -n <app-name> deployment/<app-name> --previous

# Describe the Pod for scheduling errors, image pull failures
kubectl describe pod -n <app-name> <pod-name>

# Flux kustomization status
kubectl get kustomization -n flux-system
kubectl describe kustomization apps -n flux-system
```

---

## Removing an App

Remove the app's line from `apps/staging/kustomization.yaml`, commit and push. Because `prune: true` is set on the Flux Kustomization, Flux will garbage-collect every resource it previously applied for that app — Pods, Services, Ingress, PVC, Namespace — on the next reconcile.

If you want to keep the PVC (to preserve data), delete the PVC from storage.yaml first, push that, wait for Flux to reconcile, then remove the app from the aggregate kustomization.

---

## Troubleshooting

### Flux is not picking up my changes

Check what SHA Flux is actually on versus what is in GitHub:

```bash
kubectl get gitrepository flux-system -n flux-system -o jsonpath='{.status.artifact.revision}'
git ls-remote origin HEAD
```

If they differ, force a source reconcile:

```bash
flux reconcile source git flux-system
```

If they still differ, check the GitRepository URL — it must point to the correct repo:

```bash
kubectl get gitrepository flux-system -n flux-system -o jsonpath='{.spec.url}'
```

### Flux reconciliation is failing

```bash
kubectl describe kustomization apps -n flux-system | grep -A 10 "Conditions:"
```

Common causes:
- **SOPS decryption failed** — the `sops-age` secret in `flux-system` does not match the age key used to encrypt your secrets. Verify with `kubectl get secret sops-age -n flux-system -o jsonpath='{.data.age\.agekey}' | base64 -d | grep "public key"` and compare against the recipient in your encrypted files.
- **Invalid manifest** — a YAML error in one of your files. The error message names the resource and field.
- **Context deadline exceeded** — the reconciliation is taking longer than the 5-minute timeout. Usually caused by a resource stuck in a failing state upstream.

### Pod is crash-looping

```bash
kubectl logs -n <app-name> deployment/<app-name> --previous
```

Common causes:
- **Permission denied on mounted volume** — check whether the image needs `fsGroup` in the security context, or whether the image manages its own permissions and `runAsUser` is interfering.
- **Missing environment variable or secret** — check the container logs for the specific missing value.
- **Wrong mount path** — the app is looking for its data in a path that does not match your `mountPath`.

### Service has no endpoints

```bash
kubectl get endpoints -n <app-name> <app-name>
```

If `ENDPOINTS` is `<none>`, the Service selector does not match the Pod labels. Compare `spec.selector` in the Service with `spec.template.metadata.labels` in the Deployment — they must be identical.

---

## File Checklist

```
apps/base/<app-name>/
  ✓ namespace.yaml
  ✓ deployment.yaml       image tag pinned, volume wiring correct
  ✓ service.yaml          selector matches Deployment pod labels
  ✓ storage.yaml          (if stateful)
  ✓ kustomization.yaml    lists all files above, no namespace field

apps/staging/<app-name>/
  ✓ kustomization.yaml    namespace set, references base, lists ingress.yaml
  ✓ ingress.yaml          ingressClassName: traefik, service name and port match

apps/staging/kustomization.yaml
  ✓ <app-name> added to resources list
```
