# Post Mortem: MetalLB VIP Unreachable — WiFi Promiscuous Mode (10.0.0.201)

**Date:** 2026-05-13  
**Duration:** ~90 minutes (single investigation session)  
**Severity:** Medium (Uptime Kuma, Longhorn UI, Linkding, Grafana all unreachable — all ingress traffic blocked)  
**Status:** Resolved

---

## Summary

All services behind the Traefik ingress at MetalLB VIP `10.0.0.201` were unreachable. ARP for the VIP failed from every client machine, yet the pods were healthy and NodePort access worked. The root cause was that WiFi firmware in managed (infrastructure) mode silently discards ARP broadcast requests for IP addresses it doesn't own. MetalLB's ARP responder raw socket never received incoming requests, so it could never reply. A secondary issue — stale `ServiceL2Status` CRs with immutable field errors — blocked re-announcement after a speaker restart.

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| ~22:45 | Uptime Kuma and Longhorn UI reported unreachable |
| ~23:00 | Investigation begins — `ping 10.0.0.201` returns "Destination Host Unreachable", `ip neigh` shows ARP FAILED |
| ~23:10 | Pods confirmed healthy; services confirmed working via NodePort (HTTP 302 / 401) |
| ~23:15 | MetalLB speaker logs show `serviceAnnounced` for 10.0.0.201 — MetalLB thinks it's announcing |
| ~23:20 | `arping` from client machine fails; `arping` from picomaster node also fails |
| ~23:30 | tcpdump started on picoworker02 (the announcing node) — arping immediately starts working |
| ~23:32 | ARP confirmed working between nodes; VIP reachable only while tcpdump is active |
| ~23:35 | Root cause identified: tcpdump forces `wlan0` into promiscuous mode; without it the WiFi driver drops foreign-IP ARP broadcasts before MetalLB's socket sees them |
| ~23:38 | `ip link set wlan0 promisc on` applied to picoworker02 via privileged pod; VIP becomes reachable |
| ~23:45 | MetalLB speakers restarted to clear duplicate `ServiceL2Status` entries; announcements stop |
| ~23:50 | Second root cause identified: new speaker pods fail with `status.node: Invalid value: immutable` on `l2-s5mkc` — MetalLB bug triggers on speaker restart |
| ~23:55 | All `ServiceL2Status` objects deleted; services annotated to trigger re-reconciliation; announcements resume |
| ~00:05 | `wlan0-promisc` DaemonSet deployed to all three nodes; promisc mode persistent |
| ~00:10 | Full connectivity confirmed; committed to git (`7d66dea`) |

---

## Root Causes

### 1. WiFi firmware ARP filtering (primary)

All three Pi nodes connect over WiFi (`wlan0`). `eth0` is physically unplugged. WiFi NICs in managed (client) mode implement an optimisation at the firmware/driver level: they filter incoming frames and only pass to the kernel frames addressed to the node's own MAC, broadcast frames for well-known protocols, and multicast subscriptions. ARP broadcasts for IP addresses the card does not own (such as MetalLB's VIP) are handled by the firmware and never delivered to the kernel's socket layer.

MetalLB's Layer2 ARP responder opens an `AF_PACKET` raw socket (`ETH_P_ARP / 0x0806`) on the interface and waits to receive ARP requests so it can reply. Because the firmware filtered those frames, the socket received nothing and never sent a reply — even though MetalLB logs correctly showed the service as announced.

The diagnostic proof was that running `tcpdump` on the same interface fixed the problem immediately. `tcpdump` opens an `AF_PACKET` socket with promiscuous mode (`IFF_PROMISC`), which instructs the driver to deliver all frames to the host. With promisc active, MetalLB's socket received ARP requests and responded normally.

This issue is present on all three nodes because the VIP announcer role can move to any node after a MetalLB election.

### 2. Stale `ServiceL2Status` CRs with immutable field error (secondary)

MetalLB records which node is announcing which service in `ServiceL2Status` custom resources. The `status.node` field is immutable once set. When MetalLB speaker pods are restarted (rolling restart triggered during investigation), newly elected nodes attempt to update `status.node` to reflect the new election winner. This hits a Kubernetes validation error:

```
ServiceL2Status.metallb.io "l2-s5mkc" is invalid:
status.node: Invalid value: "string": Value is immutable
```

The reconciler enters a tight error loop, never reaches the code path that emits `serviceAnnounced`, and so no ARP is sent. This is a MetalLB bug (present in v0.14.9) that was also hit in the April 30th incident with a different CR name.

---

## Contributing Factors

- **Architecture change:** Since the April 30th postmortem, Uptime Kuma, Longhorn UI, Linkding, and Grafana all moved behind Traefik ingress. A single VIP (`10.0.0.201`) now carries all ingress traffic. The impact radius of the VIP being unreachable grew from "monitoring dashboard" to "all user-facing services".
- **Speaker restart amplified the secondary issue.** The speaker restart was triggered during investigation to force a fresh ARP announcement — a reasonable recovery step. However, it exposed the `ServiceL2Status` immutability bug, which then silenced all announcements. The investigation took a two-step recovery path that would have been avoidable with a known workaround.
- **WiFi clusters are not a supported/tested MetalLB configuration.** The MetalLB docs focus on Ethernet. The promiscuous mode requirement is an undocumented interaction between the Linux WiFi driver model and MetalLB's raw-socket ARP approach.

---

## Resolution

| Fix | Detail |
|-----|--------|
| Enable promiscuous mode on `wlan0` — permanent | `wlan0-promisc` DaemonSet in `infrastructure/configs/staging/metallb/`; runs `ip link set wlan0 promisc on` every 60s on all nodes via privileged hostNetwork pod |
| Clear stuck `ServiceL2Status` objects | `kubectl delete servicel2status -n metallb-system --all` |
| Re-trigger ARP announcements after clearing | `kubectl annotate svc traefik -n kube-system metallb.io/refresh="$(date +%s)"` and same for `uptime-kuma/uptime-kuma` |

---

## Lessons Learned

- **MetalLB Layer2 on WiFi requires promiscuous mode on the announcing interface.** The WiFi driver's ARP offloading/filtering optimisation is incompatible with MetalLB's raw socket model. This must be part of any WiFi-based cluster setup. It does not cause errors — it fails silently, which makes it very hard to diagnose.
- **tcpdump as an accidental fix is a strong diagnostic signal.** If a problem disappears when tcpdump is running on an interface, promiscuous mode is almost certainly involved. This should be the first thing checked on any WiFi node that inexplicably stops responding to ARP.
- **Never restart MetalLB speakers without knowing the `ServiceL2Status` workaround.** Restarting speakers on v0.14.9 will reliably trigger the immutable field bug. The safe recovery sequence is: delete all `ServiceL2Status` objects first, then restart, then annotate services to force re-reconciliation.
- **Blast radius of single-VIP ingress.** Routing all services through one Traefik VIP is operationally clean but means a MetalLB failure takes down everything simultaneously. A VIP failure mode that previously affected one monitoring service now affects all user-facing services.

---

## Action Items

| Action | Priority |
|--------|----------|
| Add MetalLB speaker restart runbook to cluster docs — include `ServiceL2Status` delete + service re-annotation steps | High |
| Consider adding an alert (Alertmanager) for MetalLB `ServiceL2Status` reconciler errors | Medium |
| Investigate Ethernet wiring for the Pi nodes to eliminate the WiFi ARP problem at the hardware level | Low |
| Track MetalLB issue for `ServiceL2Status` immutable field bug — check if fixed in v0.15+ | Low |
