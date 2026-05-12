# Post Mortem: Flux Reconciliation Failure — Missing sops-age Secret & Stuck Terminating Namespaces

**Date:** 2026-05-12  
**Duration:** ~20 minutes (active remediation)  
**Severity:** Medium (5 of 7 kustomizations not reconciling; cluster workloads unaffected)  
**Status:** Resolved

---

## Summary

After a cluster re-bootstrap (or node reimaging), Flux failed to reconcile five kustomizations. The root causes were: (1) the `sops-age` secret was absent from `flux-system`, preventing SOPS-encrypted secrets from being decrypted; and (2) two namespaces (`uptime-kuma`, `longhorn-system`) were stuck in `Terminating` due to resources holding orphaned finalizers, blocking Flux from applying their manifests.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-05-12 | Cluster re-bootstrapped; `kubectl get secret sops-age -n flux-system` returns NotFound |
| 2026-05-12 | `apps`, `infrastucture-controllers`, `monitoring-configs` kustomizations all `False` with "secrets sops-age not found" |
| 2026-05-12 | `sops-age` secret recreated from `age.agekey` on disk |
| 2026-05-12 | `apps` and `monitoring-configs` go `True`; `infrastucture-controllers` still blocked on `Namespace/longhorn-system Terminating` |
| 2026-05-12 | MetalLB load-balancer-cleanup finalizer stripped from `uptime-kuma` service |
| 2026-05-12 | Longhorn admission webhook configs deleted; `longhorn.io` finalizers stripped from 5 CRD resources |
| 2026-05-12 | `longhorn-system` namespace cleared; all 7 kustomizations `True` |

---

## Root Causes

**1. sops-age secret not persisted across bootstrap**

The `sops-age` secret in `flux-system` holds the AGE private key Flux uses to decrypt SOPS-encrypted manifests. It is not part of the GitOps manifests (for obvious reasons) and must be manually recreated after any cluster wipe or re-bootstrap. This was missed during the most recent bootstrap.

**2. uptime-kuma namespace stuck in Terminating**

The `uptime-kuma` Service of type `LoadBalancer` held the `service.kubernetes.io/load-balancer-cleanup` finalizer. MetalLB adds this finalizer when it allocates a VIP and removes it when the service is deleted — but only if the MetalLB controller is running at deletion time. MetalLB had been removed before the namespace was deleted, leaving the finalizer permanently orphaned.

**3. longhorn-system namespace stuck in Terminating**

Longhorn CRD resources (`engineimage/ei-db6c2b6f`, three `node.longhorn.io` objects, `backuptarget/default`) held `longhorn.io` finalizers. When the namespace was deleted, Kubernetes tried to call the Longhorn admission webhook (`validator.longhorn.io`) to validate the deletion of these resources — but the webhook's backing service had already been removed. Every deletion attempt returned `service "longhorn-admission-webhook" not found`, preventing the resources from being garbage-collected and thus preventing the namespace from terminating.

---

## Contributing Factors

- The `sops-age` secret is a manual bootstrap step with no automated check. There is no Flux alert or health check that fires specifically on "SOPS key missing" versus generic reconciliation failure.
- Longhorn's admission webhook is registered cluster-wide (`ValidatingWebhookConfiguration`, `MutatingWebhookConfiguration`) and survives the deletion of the Longhorn namespace. When Longhorn is uninstalled by simply deleting the namespace rather than via its uninstall procedure, the webhooks persist and block any future deletion of lingering Longhorn CRD resources.

---

## Resolution

| Fix | Command |
|-----|---------|
| Recreate sops-age secret | `kubectl create secret generic sops-age --namespace=flux-system --from-file=age.agekey=./age.agekey` |
| Strip MetalLB finalizer from service | `kubectl patch service uptime-kuma -n uptime-kuma -p '{"metadata":{"finalizers":[]}}' --type=merge` |
| Delete stale Longhorn webhook configs | `kubectl delete validatingwebhookconfiguration longhorn-webhook-validator` <br>`kubectl delete mutatingwebhookconfiguration longhorn-webhook-mutator` |
| Strip Longhorn finalizers from CRD resources | See runbook below |
| Force Flux reconcile | `flux reconcile kustomization infrastucture-controllers -n flux-system` <br>`flux reconcile kustomization infrastructure-configs -n flux-system` |

---

## Runbook: Recovering from This State

### Step 1 — Diagnose missing sops-age secret

```bash
kubectl get secret sops-age -n flux-system
kubectl get kustomization -n flux-system
```

If kustomizations show `secrets "sops-age" not found`:

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=./age.agekey
```

The `age.agekey` file lives at the repo root (gitignored). If you are on a fresh machine without it, retrieve it from your secure backup before proceeding.

---

### Step 2 — Identify stuck terminating namespaces

```bash
kubectl get namespace | grep Terminating
```

For each stuck namespace, inspect conditions:

```bash
kubectl get namespace <name> -o json | \
  python3 -c "import sys,json; ns=json.load(sys.stdin); \
  [print(c['type'],':',c['message']) for c in ns['status'].get('conditions',[])]"
```

---

### Step 3 — Fix MetalLB load-balancer-cleanup finalizer

Applies when `NamespaceFinalizersRemaining` mentions `service.kubernetes.io/load-balancer-cleanup`.

```bash
# Find the stuck service
kubectl get services -n <namespace>

# Strip its finalizer
kubectl patch service <service-name> -n <namespace> \
  -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

### Step 4 — Fix Longhorn stuck namespace

Applies when `NamespaceDeletionContentFailure` mentions `longhorn-admission-webhook` not found.

```bash
# 1. Delete the stale webhook configurations
kubectl delete validatingwebhookconfiguration longhorn-webhook-validator
kubectl delete mutatingwebhookconfiguration longhorn-webhook-mutator

# 2. Find remaining resources with longhorn.io finalizers
kubectl get engineimages.longhorn.io -n longhorn-system -o name
kubectl get nodes.longhorn.io -n longhorn-system -o name
kubectl get backuptargets.longhorn.io -n longhorn-system -o name
kubectl get volumes.longhorn.io -n longhorn-system -o name
kubectl get replicas.longhorn.io -n longhorn-system -o name

# 3. Strip finalizers from each resource found
kubectl patch <resource>/<name> -n longhorn-system \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# 4. Verify namespace is gone
kubectl get namespace longhorn-system
```

If the namespace is still stuck after stripping all known resource finalizers, check for any remaining resources:

```bash
kubectl get namespace longhorn-system -o json | \
  python3 -c "import sys,json; ns=json.load(sys.stdin); \
  [print(c['message']) for c in ns['status'].get('conditions',[]) \
  if 'Remaining' in c['type']]"
```

---

### Step 5 — Force Flux reconciliation

Once namespaces have cleared, Flux kustomizations may still show stale error status. Trigger a manual reconcile:

```bash
flux reconcile kustomization infrastucture-controllers -n flux-system
flux reconcile kustomization infrastructure-configs -n flux-system
flux reconcile kustomization apps -n flux-system

# Verify all are True
kubectl get kustomization -n flux-system
```

---

## Lessons Learned

- **The `sops-age` secret must be the very first step after any re-bootstrap.** Add it to the cluster bootstrap checklist before applying any other Flux configuration.
- **Never uninstall Longhorn by deleting the namespace directly.** Use the official Longhorn uninstall procedure (set `deleting-confirmation-flag` to `true` via the Longhorn UI or the uninstall job) so the controller removes its own finalizers and webhook configurations cleanly.
- **MetalLB must be running when you delete LoadBalancer services.** If removing MetalLB, delete all `LoadBalancer` services first, wait for MetalLB to clear their finalizers, then uninstall MetalLB.

---

## Action Items

| Action | Priority |
|--------|----------|
| Add sops-age secret creation to cluster bootstrap documentation | High |
| Add Longhorn proper uninstall procedure to journal | Medium |
| Consider a Flux `Alert` resource that fires on repeated reconciliation failures | Low |
