# Post Mortem: Grafana CrashLoopBackOff — Duplicate Default Datasource

**Date:** 2026-05-26
**Duration:** ~3 days 8 hours (undetected until manual observation)
**Severity:** Medium (Grafana unavailable — no dashboards or log exploration; Prometheus and Alertmanager unaffected)
**Status:** Resolved

---

## Summary

Grafana entered CrashLoopBackOff immediately after the loki-stack Helm release was deployed. The pod restarted 8+ times over 3 days before the issue was investigated. The root cause was that the loki-stack Helm chart (v2.10.2) creates a Grafana datasource ConfigMap with `isDefault: true` regardless of whether its bundled Grafana is enabled. Grafana's sidecar discovered this ConfigMap alongside the existing Prometheus datasource — also marked `isDefault: true` — and Grafana exited non-zero at startup because only one datasource per organisation can be the default. A secondary issue was discovered during remediation: the Loki StatefulSet had been provisioned with `storage: "0"` due to a blank `persistence.size` value in the HelmRelease, meaning Loki had never successfully stored logs.

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| ~2026-05-23 | loki-stack HelmRelease deployed; Grafana immediately begins CrashLoopBackOff |
| ~2026-05-23 | No alert fires — Grafana downtime goes unnoticed for 3+ days |
| 2026-05-26 ~23:00 | CrashLoopBackOff noticed manually: `2/3 Running, 8 restarts, 3d8h` |
| ~23:05 | `kubectl describe pod` run — events show `FailedMount` (transient) and `BackOff restarting failed container grafana` |
| ~23:08 | `kubectl logs --previous -c grafana` run — datasource provisioning error found: `Only one datasource per organisation can be marked as default` |
| ~23:10 | All labeled datasource ConfigMaps listed — two found with `isDefault: true`: `kube-prometheus-stack-grafana-datasource` (Prometheus) and `loki-stack` (Loki) |
| ~23:12 | Root cause confirmed: loki-stack chart creates datasource ConfigMap with `isDefault: true` even with `grafana.enabled: false` |
| ~23:14 | `loki-stack` ConfigMap patched directly: `isDefault: false` |
| ~23:15 | `kubectl rollout restart deployment kube-prometheus-stack-grafana` — pod recovers to `3/3 Running` |
| ~23:18 | Permanent fix added: `postRenderers` JSON patch in `release.yaml` to enforce `isDefault: false` on every Flux reconcile |
| ~23:20 | `flux reconcile helmrelease loki-stack` triggered — upgrade fails: `StatefulSet volumeClaimTemplates` immutable field error |
| ~23:22 | Loki StatefulSet inspected — `storage: "0"` found; no PVC ever provisioned |
| ~23:25 | `persistence.size: 5Gi` and `retention_period: 168h` values set; `upgrade.force: true` added to HelmRelease |
| ~23:28 | Broken StatefulSet deleted; HelmRelease reconciled — Loki recreated with `5Gi` PVC |
| ~23:30 | Loki verified: `/ready` returns `ready`, 12 active log streams, 7 namespaces ingested |
| ~23:35 | Logs confirmed visible in Grafana Explore via LogQL |

---

## Root Causes

### 1. loki-stack chart creates datasource ConfigMap unconditionally (primary)

The loki-stack Helm chart v2.10.2 includes a template that creates a Grafana datasource ConfigMap labeled `grafana_datasource: "1"`. This label is how Grafana's `grafana-sc-datasources` sidecar discovers and loads datasources at runtime — it watches for any ConfigMap with that label across the cluster. The loki-stack chart creates this ConfigMap with `isDefault: true` for the Loki datasource.

The HelmRelease had `grafana.enabled: false` to avoid deploying a second Grafana instance alongside kube-prometheus-stack's Grafana. This flag disables the Grafana deployment, but does not suppress the datasource ConfigMap. The ConfigMap was created by the chart and picked up by the existing Grafana sidecar.

`kube-prometheus-stack` had already configured Prometheus as the default datasource (`isDefault: true`). With two defaults present, Grafana's provisioner exits non-zero, and the container crashes on every startup attempt.

The datasource for Loki had already been correctly defined in `loki-datasource-patch.yaml` (part of the kube-prometheus-stack overlay) with `isDefault: false`. The loki-stack chart's ConfigMap was a duplicate and a conflict.

### 2. Loki StatefulSet provisioned with `storage: "0"` (secondary)

The `persistence.size` field in `release.yaml` was left blank (a `TODO(human)` placeholder). The loki-stack chart rendered a blank value as `storage: "0"` in the StatefulSet's `volumeClaimTemplates`. Kubernetes accepted this and created the StatefulSet, but the PVC was never bound — Loki was running but writing nowhere. This meant the cluster had no persistent log storage for the entire deployment period.

---

## Contributing Factors

- **No Grafana availability alert.** Grafana was down for over 3 days before it was noticed by manual observation. There was no Alertmanager rule firing on `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` for the monitoring namespace. The irony: the monitoring tool that would display this alert was the thing that was down.
- **Blank placeholder values committed.** `TODO(human)` placeholders in the HelmRelease were committed and pushed without being filled in. The chart silently accepted the blank value and rendered a broken StatefulSet rather than erroring at install time.
- **Helm upgrade blocked by immutable StatefulSet field.** Changing `volumeClaimTemplates.storage` requires deleting and recreating the StatefulSet — Kubernetes does not allow patching it. Helm's default upgrade path tries to patch, hits the rejection, and leaves the release in a failed state. Without `upgrade.force: true`, this is a manual intervention every time this field changes.

---

## Resolution

| Fix | Detail |
|-----|--------|
| Patch `loki-stack` ConfigMap to `isDefault: false` | `kubectl patch configmap loki-stack -n monitoring` — immediate unblock |
| Add `postRenderers` to loki-stack HelmRelease | JSON patch overwrites `isDefault: true` in the chart-rendered ConfigMap on every Flux reconcile — permanent fix, survives upgrades |
| Set `persistence.size: 5Gi`, retention `168h` | Fills in all four matching retention fields in `release.yaml` |
| Add `upgrade.force: true` to HelmRelease | Allows Helm to delete and recreate resources with immutable fields rather than failing the upgrade |
| Delete broken StatefulSet (`storage: "0"`) | `kubectl delete statefulset loki-stack -n monitoring` — allows Flux to recreate with correct `5Gi` PVC |

---

## Lessons Learned

<!-- TODO(human): Write 3–5 lessons from your own perspective.
     What did this incident change about how you think?
     Consider: what you'd do differently when deploying a chart with grafana.enabled: false,
     how you think about placeholder values in GitOps configs,
     and what the 3-day blind spot says about your alerting coverage. -->

---

## Action Items

| Action | Priority |
|--------|----------|
| Add Alertmanager rule for `CrashLoopBackOff` in the monitoring namespace | High |
| Add comment in `release.yaml` next to `upgrade.force: true` explaining why it's needed | Medium |
| Review all other HelmReleases for blank placeholder values that may have been committed | Medium |
| Document the `grafana_datasource: "1"` label behaviour in cluster journal — any new chart that ships this label needs its `isDefault` checked | Medium |
