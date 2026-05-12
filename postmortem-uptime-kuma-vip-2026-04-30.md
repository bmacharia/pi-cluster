# Post Mortem: uptime-kuma VIP Unreachable (10.0.0.201:3001)

**Date:** 2026-04-30 — 2026-05-04
**Duration:** ~4 days (intermittent investigation)
**Severity:** Low (monitoring service down, no production workloads affected)
**Status:** Resolved

---

## Summary

The uptime-kuma service was unreachable at its MetalLB-assigned VIP `10.0.0.201:3001`. ARP for the VIP was permanently incomplete on the network, meaning no machine outside the cluster could route to it. The underlying pod and NodePort access were healthy throughout.

---

## Timeline

| Time | Event |
|------|-------|
| Apr 30 | Issue discovered — `ping 10.0.0.201` returns "Destination Host Unreachable", ARP incomplete |
| Apr 30 | MetalLB memberlist logs identified: "TCP connects but UDP probes failed" on all nodes |
| Apr 30 | Root cause 1 identified: k3s klipper-lb (built-in ServiceLB) not disabled — competing with MetalLB for the service |
| Apr 30 | Root cause 2 identified: UDP port 7946 blocked by iptables on all nodes (Ubuntu 24.04 uses nftables backend; k3s uses iptables-legacy — rules added to wrong backend initially) |
| Apr 30 | Root cause 3 identified: stale `ServiceL2Status` CR stuck in reconciliation loop — `status.node` field is immutable, preventing MetalLB from updating node ownership |
| Apr 30 | Fixes applied; investigation paused |
| May 4 | Resumed — tcpdump confirmed UDP 7946 memberlist traffic flowing correctly between all nodes |
| May 4 | `arping` confirmed MetalLB's ARP responder working; VIP accessible via HTTP |

---

## Root Causes

**1. k3s built-in ServiceLB (klipper-lb) not disabled**

k3s ships with its own LoadBalancer implementation. When MetalLB is installed alongside it, both attempt to manage `LoadBalancer` type services. This caused `svclb-uptime-kuma-*` pods to coexist with MetalLB, creating conflicting iptables NAT rules and ICMP redirect behaviour. k3s was only configured to disable `helm-controller`, not `servicelb`.

**2. MetalLB memberlist UDP blocked by iptables**

MetalLB's Layer2 mode uses the Hashicorp memberlist gossip protocol on UDP/TCP port 7946 to elect which node announces the VIP via ARP. On Ubuntu 24.04, `iptables` uses the nftables backend while k3s uses `iptables-legacy`. Rules added with `iptables` were invisible to k3s's packet path. Additionally, k3s does not open port 7946 by default. Without functioning UDP memberlist, MetalLB could not elect a stable ARP-announcing node, causing the VIP to flap between nodes every few seconds and never settle.

**3. Stale `ServiceL2Status` custom resource**

A `ServiceL2Status` CR (`l2-wzqv9`) had `status.node` set to an immutable value from a previous election. When the unstable memberlist caused node ownership to change, the controller repeatedly tried to update this field, hit a validation error, and re-queued — creating a tight reconciliation loop that prevented a clean ARP announcement.

---

## Contributing Factors

- **All cluster nodes and the client machine are on WiFi (wlan0).** WiFi APs can suppress ARP broadcasts between wireless clients. This masked the fact that MetalLB's ARP responder was actually functioning — the ARP replies were unicast and reached the client correctly once explicitly probed via `arping`, but the OS ARP cache never populated from gratuitous ARPs alone.
- **Two iptables backends co-existing on Ubuntu 24.04** made diagnosis harder — rules appeared to be in place but were silently in the wrong table.
- **MetalLB's "UDP probes failed" log message** correctly identified the problem but was still appearing in logs from old pod instances, causing confusion about whether fixes had taken effect.

---

## Resolution

| Fix | Command |
|-----|---------|
| Disable k3s klipper-lb | Added `'--disable=servicelb'` to `/etc/systemd/system/k3s.service` on picomaster; restarted k3s |
| Open port 7946 on all nodes | `iptables-legacy -I INPUT -p udp --dport 7946 -j ACCEPT` + TCP equivalent; saved with `netfilter-persistent` |
| Delete stuck L2Status CR | `kubectl delete servicel2status l2-wzqv9 -n metallb-system` |

---

## Lessons Learned

- **When installing MetalLB on k3s, always explicitly disable the built-in ServiceLB** with `--disable=servicelb`. The k3s docs mention this but it is easy to miss.
- **Ubuntu 24.04 + k3s = dual iptables backends.** Any manual firewall rules must be applied to `iptables-legacy`, not `iptables`, since k3s uses the legacy backend. Consider using `iptables-persistent` with `iptables-legacy-save` specifically.
- **Port 7946 (MetalLB memberlist) must be explicitly opened** on all cluster nodes. This should be part of the standard node provisioning playbook.
- **WiFi clusters have ARP limitations.** MetalLB Layer2 mode works on WiFi but gratuitous ARPs may not populate remote caches. This is not a blocker but is worth noting in runbooks — if a VIP is announced but unreachable, `arping` from the client machine will force resolution.

---

## Action Items

| Action | Priority |
|--------|----------|
| Add port 7946 TCP/UDP to node provisioning (Ansible/cloud-init) | High |
| Document MetalLB + k3s setup requirements in cluster runbook | Medium |
| Consider moving cluster nodes to wired ethernet to eliminate WiFi ARP edge cases | Low |
