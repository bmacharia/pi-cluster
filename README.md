# pi-cluster

A 3-node K3s cluster running on Raspberry Pis at home. I use it to learn Kubernetes the hard way — by running real workloads on it, breaking things, and writing postmortems when I do.

The cluster is managed declaratively through GitOps: this repo is the source of truth, and FluxCD reconciles the cluster toward it. Everything you'd change with `kubectl` lives here instead.

## For Hiring Managers

**What this demonstrates:** Production-grade CI/CD pipeline design — from code commit to deployed application, with automated quality gates at every stage.

**What to evaluate:**
- `.github/workflows/` — GitHub Actions pipeline definitions
- `kubernetes/` — Kustomize overlays for environment-specific deployment
- `.devcontainer/` — Standardized developer environment
- `src/` — Application code (Python 3.13)
- `release-please-config.json` — Automated semantic versioning

**Skills demonstrated:** GitHub Actions, Trivy, PyTest, Ruff, Release Please, DevContainers, K3d, Kustomize, Docker, Python, zero-downtime deployment patterns

**Time invested:** 47+ commits


## What's running

- **K3s** on three Raspberry Pis (one server, two agents), all on Wi-Fi
- **FluxCD** for GitOps reconciliation, with **SOPS + Age** for encrypted secrets in Git
- **MetalLB** (Layer 2) for `LoadBalancer` services
- **Traefik** as ingress, with **Cloudflare Tunnel** for external access
- **Longhorn** for persistent storage
- **Prometheus, Grafana, Alertmanager** for monitoring
- **Renovate** for automated dependency updates
- Workloads: Uptime Kuma, Linkding, Grafana

## Repo layout

```
apps/             # application workloads (base + staging overlays)
clusters/staging/ # Flux bootstrap manifests for the cluster
infrastructure/   # MetalLB, Traefik, Longhorn, cert-manager, etc.
monitoring/       # Prometheus / Grafana / Alertmanager
journal/          # runbooks and operational notes
postmortem-*.md   # incident postmortems (the interesting bits)
```

## How a change reaches the cluster

1. Push a commit to `main`.
2. Flux's `source-controller` notices the new revision.
3. `kustomize-controller` builds the kustomization, decrypts any SOPS-encrypted secrets, and diffs against the live cluster.
4. Changes are applied. If reconciliation fails, the previous state is preserved and the failure surfaces in `flux get kustomizations`.

No `kubectl apply` from a laptop, ever. If it isn't in Git, it isn't in the cluster.

## Postmortems

The most useful artifacts in this repo. Each one is a real outage I debugged on this cluster:

- [**MetalLB VIP unreachable — Wi-Fi promiscuous mode**](./postmortem-metallb-vip-wifi-promisc-2026-05-13.md) — All ingress went dark. MetalLB logs said "announced." ARP failed everywhere. `tcpdump` accidentally fixed it, which turned out to be the diagnostic: Wi-Fi firmware filters incoming ARP for IPs the card doesn't own, so MetalLB's raw socket never saw the requests. Permanent fix is a DaemonSet that keeps `wlan0` in promiscuous mode.
- [**Flux reconciliation failure — missing sops-age secret & stuck Terminating namespaces**](./postmortem-flux-sops-stuck-namespaces-2026-05-12.md) — Post-bootstrap, five kustomizations wouldn't reconcile. SOPS key was missing (it's not in Git, by design) and two namespaces were stuck Terminating because Longhorn's cluster-wide admission webhooks outlived its namespace, blocking finalizer cleanup.
- [**uptime-kuma VIP unreachable**](./postmortem-uptime-kuma-vip-2026-04-30.md) — Three overlapping causes: k3s's built-in `klipper-lb` was still running alongside MetalLB, MetalLB's memberlist port 7946 was blocked because Ubuntu 24.04 ships dual iptables backends and rules went to the wrong one, and a stale `ServiceL2Status` CR was stuck in a reconciliation loop on an immutable field.

## What this isn't

It's a homelab. The control plane is a single node, storage is Longhorn on SD cards, and the nodes talk over Wi-Fi. A real production cluster would have HA control plane, wired networking (ideally with BGP-mode MetalLB), proper persistent storage, secret management backed by a KMS, and a CI pipeline gating PRs. The point of this project is to operate something end-to-end, not to pretend it's enterprise infrastructure.

## Bootstrap

```
flux bootstrap github \
  --owner=bmacharia \
  --repository=pi-cluster \
  --branch=main \
  --path=./clusters/staging
```

Then create the SOPS key secret (this step is intentionally outside Git):

```
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=./age.agekey
```

---

**Maintainer:** Babu Macharia · [linkedin.com/in/babu-macharia](https://linkedin.com/in/babu-macharia) · [babumacharia.com](https://babumacharia.com)
