# Kubernetes Debugging for SREs — Lesson 01 Lab Pack

## Lesson: The Layered Debugging Approach

This lab pack is designed for your current setup:

```text
3-node K3s cluster
FluxCD-managed GitOps platform
apps/staging = stable applications
apps/test = safe lab environment
sre-debug namespace = Kubernetes debugging practice namespace
```

The goal of this lesson is not to master every Kubernetes failure mode yet. The goal is to build the SRE debugging reflex:

```text
Scope first.
Events second.
Recent change third.
Layer hypothesis before deep debugging.
```

The central rule for this lesson:

```text
Locate the failure layer first. Investigate the specific cause second.
```

---

# 1. The Five Debugging Layers

Every incident should first be mapped to one likely layer:

| Layer | What It Means | Common Signals |
|---|---|---|
| Application | App code, config, dependency, runtime behavior | Pods healthy, app returns errors, logs show app failure |
| Pod | Kubernetes lifecycle failure | CrashLoopBackOff, OOMKilled, ImagePullBackOff, Evicted, Pending |
| Node | Node health or capacity issue | Node NotReady, DiskPressure, MemoryPressure, pods failing on one node |
| Cluster | Shared cluster service failure | CoreDNS, API server, scheduler, etcd, kube-proxy/CNI symptoms |
| Cloud / Infrastructure | Underlying provider or infra failure | IAM, quota, storage attach, AZ outage, network provider issue |

For your homelab, “Cloud” often maps to:

```text
LAN
DNS
MetalLB
Longhorn
Node hardware
Power/network issues
Cloudflare tunnel
Router/firewall
```

---

# 2. Two-Minute Triage Framework

For every lab, start with this exact sequence before jumping into logs.

## Check 1: Scope of impact

```bash
kubectl get pods -A -o wide
kubectl get pods -n sre-debug -o wide
```

Ask:

```text
Is this affecting one workload, one namespace, one node, or the whole cluster?
```

## Check 2: Recent cluster events

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Ask:

```text
What is Kubernetes trying to tell me?
```

Look for:

```text
FailedScheduling
BackOff
Failed
Pulled / Pulling
Unhealthy
NodeNotReady
Killing
Evicted
```

## Check 3: What changed?

```bash
git log --oneline -5
flux get kustomizations -n flux-system
flux get sources git -A
```

Ask:

```text
Was there a Git change, Flux reconciliation, deploy, config update, or platform change?
```

---

# 3. Repo Structure for These Labs

Use this structure:

```text
apps/test/sre-debug-app/
├── kustomization.yaml
├── scenarios/
│   ├── 01-application-bad-config/
│   │   └── app-bad-config-patch.yaml
│   ├── 02-pod-crashloop/
│   │   └── crashloop-patch.yaml
│   ├── 03-pod-imagepullbackoff/
│   │   └── bad-image-patch.yaml
│   ├── 04-scheduling-pending/
│   │   └── too-much-cpu-patch.yaml
│   ├── 05-service-selector-break/
│   │   └── service-selector-patch.yaml
│   └── 06-cluster-dns-observation/
│       └── README.md
└── README.md
```

Your active overlay should reference only one scenario patch at a time.

Do not activate multiple failure patches at once.

---

# 4. Baseline Healthy State

Before breaking anything, confirm the lab app is healthy.

## Commands

```bash
kubectl get pods -n sre-debug -o wide
kubectl get deploy -n sre-debug
kubectl get svc -n sre-debug
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

## Service test

```bash
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -s http://sre-debug-app
```

## Expected result

```text
Pods are Running
READY is 1/1
Deployment has desired replicas available
Service exists
No recent Warning events
Curl returns a response
```

## Baseline layer conclusion

```text
No active incident. App, pod, node, and cluster service routing appear healthy.
```

---

# 5. Lab 1 — Application-Layer Bad Config

## Scenario

The pod is healthy, Kubernetes is happy, but a recent application configuration change was introduced.

This lab teaches:

```text
Running pod does not always mean healthy application behavior.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/01-application-bad-config/app-bad-config-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: APP_MODE
              value: "broken-config"
```

## Activate the patch

Edit:

```text
apps/test/sre-debug-app/kustomization.yaml
```

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/01-application-bad-config/app-bad-config-patch.yaml
```

## Commit and reconcile

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: introduce application config change"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage commands

```bash
kubectl get pods -n sre-debug -o wide
kubectl describe deploy sre-debug-app -n sre-debug
kubectl get events -A --sort-by=.lastTimestamp | tail -30
git log --oneline -5
```

Only after that, check logs:

```bash
kubectl logs -n sre-debug -l app=sre-debug-app --tail=50
```

## Expected signals

```text
Pods are Running
Pods are Ready
No restart loop
No scheduling failure
Recent Git change exists
```

## Layer hypothesis

```text
Application / configuration layer
```

## Why

Kubernetes lifecycle is healthy, but there was a recent app-level configuration change.

## Fix

Remove the patch from `kustomization.yaml`.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix: remove application config lab patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Verify recovery

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

---

# 6. Lab 2 — Pod-Layer CrashLoopBackOff

This is the lab you said you want to start with.

## Scenario

The container starts, prints a message, then exits with status code `1`. Kubernetes restarts it repeatedly.

This lab teaches:

```text
CrashLoopBackOff is a pod/container lifecycle failure, not a node or cluster outage.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/02-pod-crashloop/crashloop-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  template:
    spec:
      containers:
        - name: app
          command: ["/bin/sh", "-c"]
          args:
            - echo "Simulated crash for SRE debugging lab"; exit 1
```

## Activate the patch

Edit:

```text
apps/test/sre-debug-app/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/02-pod-crashloop/crashloop-patch.yaml
```

## Commit and reconcile

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: introduce CrashLoopBackOff"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Two-minute triage

Run these first:

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
flux get kustomizations -n flux-system
git log --oneline -5
```

Before checking logs, answer:

```text
What layer is failing?
What evidence tells me that?
```

## Deeper investigation

```bash
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl logs -n sre-debug -l app=sre-debug-app --previous --tail=50
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
```

## Expected signals

```text
CrashLoopBackOff
Back-off restarting failed container
Exit Code: 1
Previous logs show simulated crash
```

## Layer hypothesis

```text
Pod layer
```

## Why

The failure is happening at the container lifecycle level. Kubernetes can schedule the pod, but the container process exits.

## Fix

Remove the patch from `kustomization.yaml`.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix: remove CrashLoopBackOff lab patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Verify recovery

```bash
kubectl get pods -n sre-debug -o wide
kubectl rollout status deployment/sre-debug-app -n sre-debug
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

Expected:

```text
Pods return to Running
READY becomes 1/1
No new BackOff events
```

---

# 7. Lab 3 — Pod-Layer ImagePullBackOff

## Scenario

Kubernetes cannot pull the image, so the application never starts.

This lab teaches:

```text
ImagePullBackOff is a pod lifecycle failure before the app even runs.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/03-pod-imagepullbackoff/bad-image-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  template:
    spec:
      containers:
        - name: app
          image: ghcr.io/bmacharia/this-image-does-not-exist:lesson-1
```

## Activate the patch

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/03-pod-imagepullbackoff/bad-image-patch.yaml
```

## Commit and reconcile

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: introduce ImagePullBackOff"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Two-minute triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
flux get kustomizations -n flux-system
git log --oneline -5
```

## Deeper investigation

```bash
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
```

## Expected signals

```text
ErrImagePull
ImagePullBackOff
failed to pull image
repository not found or denied
```

## Layer hypothesis

```text
Pod layer
```

## Why

The container cannot be created because the image cannot be pulled. The application process never starts.

## Fix

Remove the patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix: remove ImagePullBackOff lab patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Verify recovery

```bash
kubectl get pods -n sre-debug -o wide
kubectl rollout status deployment/sre-debug-app -n sre-debug
```

---

# 8. Lab 4 — Scheduling / Pending Pod

## Scenario

The pod requests more CPU than your 3-node K3s cluster can provide, so it remains Pending.

This lab teaches:

```text
Pending is often a scheduling or capacity issue, not an application bug.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/04-scheduling-pending/too-much-cpu-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: "100"
              memory: 128Mi
            limits:
              cpu: "100"
              memory: 128Mi
```

## Activate the patch

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/04-scheduling-pending/too-much-cpu-patch.yaml
```

## Commit and reconcile

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: introduce unschedulable pod"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Two-minute triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
kubectl get nodes -o wide
flux get kustomizations -n flux-system
git log --oneline -5
```

## Deeper investigation

```bash
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl describe nodes | grep -A5 -E "Name:|Allocated resources|Conditions"
```

## Expected signals

```text
Pending
FailedScheduling
0/3 nodes are available
Insufficient cpu
```

## Layer hypothesis

```text
Node / scheduling capacity layer
```

## Why

The pod is not failing because the app crashed. It is failing because the scheduler cannot place it on any node.

## Fix

Remove the patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix: remove unschedulable pod lab patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Verify recovery

```bash
kubectl get pods -n sre-debug -o wide
kubectl rollout status deployment/sre-debug-app -n sre-debug
```

---

# 9. Lab 5 — Service Selector Break

## Scenario

Pods are healthy. The Service exists. But the Service selector no longer matches the pod labels, so traffic does not route to the pods.

This lab teaches:

```text
Healthy pods do not guarantee healthy service routing.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/05-service-selector-break/service-selector-patch.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  selector:
    app: wrong-label
```

## Activate the patch

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/05-service-selector-break/service-selector-patch.yaml
```

## Commit and reconcile

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: break service selector"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Two-minute triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -30
flux get kustomizations -n flux-system
git log --oneline -5
```

## Deeper investigation

```bash
kubectl get pods -n sre-debug --show-labels
kubectl get svc -n sre-debug sre-debug-app -o yaml
kubectl get endpoints -n sre-debug sre-debug-app
kubectl get endpointslice -n sre-debug
```

## Service test

```bash
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -m 5 http://sre-debug-app
```

## Expected signals

```text
Pods are Running
Pods are Ready
Service exists
Endpoints are empty
Curl fails or times out
```

## Layer hypothesis

```text
Cluster networking / Service discovery layer
```

## Why

The app is not crashing. The pods are healthy. The failure is between the Service and the backing pods.

## Fix

Remove the patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix: restore service selector"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Verify recovery

```bash
kubectl get endpoints -n sre-debug sre-debug-app
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -s http://sre-debug-app
```

Expected:

```text
Endpoints exist
Curl succeeds
```

---

# 10. Lab 6 — Cluster DNS Observation Drill

## Scenario

This is a safe observation drill. Do not break CoreDNS yet.

This lab teaches:

```text
Know what healthy cluster DNS looks like before debugging broken DNS.
```

## Commands

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get endpoints kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
```

## DNS test: Kubernetes service

```bash
kubectl run dns-test -n sre-debug --rm -it \
  --image=busybox:1.36 --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```

## DNS test: lab service

```bash
kubectl run dns-test -n sre-debug --rm -it \
  --image=busybox:1.36 --restart=Never -- \
  nslookup sre-debug-app.sre-debug.svc.cluster.local
```

## Expected signals

```text
CoreDNS pods Running
kube-dns Service exists
kube-dns endpoints exist
nslookup works
```

## Layer hypothesis

```text
Cluster DNS is healthy
```

## Why

DNS resolution works for both the default Kubernetes service and your lab app service.

---

# 11. Lab 7 — Anti-Pattern Drill: Do Not Start With Logs

## Scenario

Use any broken lab above, but force yourself to avoid logs for the first two minutes.

## Rule

For the first two minutes, do not run:

```bash
kubectl logs
```

Instead run:

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -30
flux get kustomizations -n flux-system
git log --oneline -5
```

Then write:

```text
Scope:
Recent events:
Recent Git change:
Likely layer:
Why:
Next command:
```

Only after that can you run:

```bash
kubectl logs
```

## Purpose

This drill trains you not to tunnel into logs before you know the failure layer.

---

# 12. Incident Note Template

Create:

```text
journal/debugging/lesson-01-layered-debugging-template.md
```

Template:

```markdown
# Lesson 01 Incident Note: <Scenario Name>

## Date

## Symptom

What looked broken?

## Scope

- One service or many services?
- Which namespace?
- Which pods?
- Which nodes?

## Events

Command:

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Important events:

```text
paste key events here
```

## Recent Change

Command:

```bash
git log --oneline -5
```

What changed?

## Layer Hypothesis

Choose one:

- Application
- Pod
- Node
- Cluster
- Cloud / Infrastructure
- GitOps / Flux

My hypothesis:

```text

```

## Evidence

Why do I think this is the layer?

## First Mitigation

What action would reduce user impact?

## Root Cause

What actually caused it?

## Fix

What Git change restored health?

## Lesson Learned

What pattern should I remember next time?
```

---

# 13. Lab Execution Rules

Use these rules every time:

```text
One lab patch at a time.
One failure at a time.
One layer hypothesis before logs.
One Git fix to recover.
One incident note after recovery.
```

Do not stack patches like this:

```yaml
patches:
  - path: scenarios/02-pod-crashloop/crashloop-patch.yaml
  - path: scenarios/03-pod-imagepullbackoff/bad-image-patch.yaml
  - path: scenarios/04-scheduling-pending/too-much-cpu-patch.yaml
```

That creates multiple simultaneous failures and weakens the training.

---

# 14. Recommended Practice Order

Use this order:

```text
1. Baseline healthy check
2. Lab 2: CrashLoopBackOff
3. Lab 3: ImagePullBackOff
4. Lab 4: Scheduling / Pending pod
5. Lab 5: Service selector break
6. Lab 6: DNS observation drill
7. Lab 7: Anti-pattern drill
8. Lab 1: Application bad config
```

Start with Lab 2 because it gives you an obvious Kubernetes lifecycle failure.

---

# 15. Quick Reference: Layer Mapping

| Symptom | Likely Layer |
|---|---|
| Pods Running, app returns wrong response | Application |
| CrashLoopBackOff | Pod |
| ImagePullBackOff | Pod |
| OOMKilled | Pod / Node pressure |
| Pending with FailedScheduling | Node / Scheduling |
| Node NotReady | Node |
| Service has no endpoints | Cluster networking / Service discovery |
| CoreDNS failing | Cluster |
| Flux source cannot pull Git | GitOps / Flux source layer |
| SOPS secret missing | GitOps / secret decryption layer |
| Longhorn volume attach failure | Infrastructure / storage layer |
| MetalLB IP conflict | Infrastructure / network layer |

---

# 16. Starting Lab 2 Now

When you are ready to start Lab 2:

```bash
mkdir -p apps/test/sre-debug-app/scenarios/02-pod-crashloop
```

Create:

```text
apps/test/sre-debug-app/scenarios/02-pod-crashloop/crashloop-patch.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  template:
    spec:
      containers:
        - name: app
          command: ["/bin/sh", "-c"]
          args:
            - echo "Simulated crash for SRE debugging lab"; exit 1
```

Edit:

```text
apps/test/sre-debug-app/kustomization.yaml
```

To include:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/02-pod-crashloop/crashloop-patch.yaml
```

Then:

```bash
git add apps/test/sre-debug-app
git commit -m "Lab: introduce CrashLoopBackOff"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

Before logs, run:

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

Then answer:

```text
What layer is failing?
What evidence tells me that?
```
