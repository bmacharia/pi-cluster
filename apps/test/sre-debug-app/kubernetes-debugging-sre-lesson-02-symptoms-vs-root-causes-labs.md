# Kubernetes Debugging for SREs — Lesson 02 Lab Pack

## Lesson 02: Symptoms vs Root Causes

This lab pack is designed for your current GitOps homelab setup:

```text
3-node K3s cluster
FluxCD-managed repo
apps/staging = stable apps
apps/test = lab environment
sre-debug namespace = safe debugging sandbox
```

The goal of this lesson is to train the discipline of separating:

```text
Symptom     = what appears broken
Mitigation  = what stops the immediate impact
Root cause  = why the symptom happened
Prevention  = what stops recurrence
```

Do not treat a restart, rollback, resource bump, or scale-up as a complete fix unless you can explain why the incident happened and what prevents it from happening again.

---

# 1. Lab Structure First

Create this structure under your existing test app:

```text
apps/test/sre-debug-app/
├── kustomization.yaml
├── scenarios/
│   └── lesson-02-symptoms-vs-root-causes/
│       ├── 01-crashloop-symptom-vs-root-cause/
│       │   └── crashloop-root-cause-patch.yaml
│       ├── 02-imagepull-mitigation-vs-fix/
│       │   └── bad-image-patch.yaml
│       ├── 03-missing-config-root-cause/
│       │   ├── missing-config-patch.yaml
│       │   └── configmap.yaml
│       ├── 04-resource-limit-symptom-fix/
│       │   ├── oom-patch.yaml
│       │   └── memory-bump-patch.yaml
│       ├── 05-service-selector-layer-mismatch/
│       │   └── bad-service-selector.yaml
│       └── 06-recurring-incident-pattern/
│           └── recurring-service-selector.yaml
└── README.md
```

Also create a journal folder for incident notes:

```text
journal/debugging/
├── lesson-02-postmortem-template.md
└── incidents/
```

If the `incidents` directory does not exist yet:

```bash
mkdir -p journal/debugging/incidents
```

---

# 2. Base Rule for All Labs

Only activate **one patch at a time**.

Your `apps/test/sre-debug-app/kustomization.yaml` should normally look like this when healthy:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app
```

When you run a lab, temporarily add one patch:

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/<scenario>/<patch-file>.yaml
```

After each lab, remove the patch, commit, push, and reconcile.

---

# 3. Universal Lab Workflow

Use this same workflow for every scenario.

## Step 1: Apply one failure patch

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce <scenario name>"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

If Flux times out because the workload is unhealthy, that may be expected. The question is whether the broken desired state was applied.

## Step 2: Triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
flux get kustomizations -n flux-system
git log --oneline -5
```

## Step 3: Answer the Lesson 2 questions

```text
Symptom:
Mitigation:
Root cause:
Cause category:
Prevention:
```

Cause categories:

```text
Bug
Configuration
Capacity
Process
Architecture
```

## Step 4: Fix through Git

Remove the active patch from `apps/test/sre-debug-app/kustomization.yaml`.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix lab 02 scenario by removing failure patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Step 5: Confirm recovery

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

Expected healthy state:

```text
sre-debug-app pods are 1/1 Running
No new warning events
test-apps eventually Ready=True
```

---

# 4. Lesson 2 Core Mental Model

## Symptom

The visible problem:

```text
Pod is CrashLoopBackOff
Pod is ImagePullBackOff
Service is unreachable
p99 latency spiked
5xx errors increased
```

## Mitigation

The immediate action that reduces impact:

```text
Rollback
Restart
Scale up
Remove bad patch
Increase resource limit
Fail over
```

## Root Cause

The underlying reason the symptom happened:

```text
Bad startup command
Wrong image tag
Missing ConfigMap
Unbounded memory allocation
Service selector does not match pod labels
```

## Prevention

A control that makes recurrence less likely:

```text
CI validation
Kustomize build check
Image tag existence check
Smoke test
Endpoint check
Resource/load test
Review checklist
Policy-as-code
```

---

# 5. Lab 1 — CrashLoopBackOff: Symptom vs Root Cause

## Learning Goal

Do not stop at:

```text
Pod is CrashLoopBackOff.
```

Find:

```text
Why is it CrashLoopBackOff?
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/01-crashloop-symptom-vs-root-cause/crashloop-root-cause-patch.yaml
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
            - echo "ROOT_CAUSE=bad_startup_command"; exit 1
```

## Activate the patch

Edit:

```text
apps/test/sre-debug-app/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/sre-debug-app

patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/01-crashloop-symptom-vs-root-cause/crashloop-root-cause-patch.yaml
```

## Apply

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce crashloop root cause scenario"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl logs -n sre-debug -l app=sre-debug-app --previous --tail=50
```

## Expected evidence

```text
CrashLoopBackOff
Back-off restarting failed container
Exit Code 1
ROOT_CAUSE=bad_startup_command
```

## Write the analysis

```text
Symptom:
sre-debug-app pod is CrashLoopBackOff.

Mitigation:
Remove the bad command patch or roll back the Git change.

Root cause:
The container command exits immediately with exit code 1.

Cause category:
Configuration or process.

Prevention:
Add review checks for command/args overrides.
Add smoke tests after Flux reconciliation.
```

## 5 Whys

```text
1. Why is the service unhealthy?
   Because the pod is CrashLoopBackOff.

2. Why is the pod CrashLoopBackOff?
   Because the container repeatedly exits.

3. Why does the container exit?
   Because the command intentionally exits with code 1.

4. Why did that command reach the cluster?
   Because the GitOps patch changed the container startup behavior.

5. What prevents recurrence?
   Review/check/policy for command overrides and post-deploy smoke testing.
```

## Fix

Remove the patch from `kustomization.yaml`.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix crashloop lab by removing bad startup command"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

---

# 6. Lab 2 — ImagePullBackOff: Mitigation vs Real Fix

## Learning Goal

Understand the difference between:

```text
Mitigation: use a working image.
Root cause: bad image reference, missing tag, registry issue, or auth issue.
```

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/02-imagepull-mitigation-vs-fix/bad-image-patch.yaml
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
          image: ghcr.io/bmacharia/sre-debug-app:not-a-real-tag
```

## Activate

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/02-imagepull-mitigation-vs-fix/bad-image-patch.yaml
```

## Apply

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce bad image tag"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
```

## Expected evidence

```text
ErrImagePull
ImagePullBackOff
failed to pull image
repository not found
manifest unknown
```

## Write the analysis

```text
Symptom:
Pod cannot become Ready.

Mitigation:
Roll back to a known-good image.

Root cause:
The image tag does not exist or the image reference is wrong.

Cause category:
Configuration or process.

Prevention:
Add image tag validation before merge.
Use Renovate/image automation.
Use digest-pinned images for production.
Run post-deploy image pull checks.
```

## 5 Whys

```text
1. Why is the app unavailable?
   Because the pod is not Ready.

2. Why is the pod not Ready?
   Because Kubernetes cannot pull the image.

3. Why can’t the image be pulled?
   Because the tag does not exist.

4. Why did the bad tag reach Git?
   Because the deployment accepted a manual image edit.

5. What prevents recurrence?
   CI image validation or automated image updates.
```

## Fix

Remove the patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix imagepull lab by removing bad image patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

---

# 7. Lab 3 — Missing Config: Configuration Root Cause

## Learning Goal

See how a missing config dependency becomes a Kubernetes symptom.

## Create the failing patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/03-missing-config-root-cause/missing-config-patch.yaml
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
            - name: REQUIRED_SETTING
              valueFrom:
                configMapKeyRef:
                  name: sre-debug-config
                  key: required-setting
```

Do **not** create the ConfigMap yet.

## Activate

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/03-missing-config-root-cause/missing-config-patch.yaml
```

## Apply

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce missing config dependency"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
```

## Expected evidence

```text
CreateContainerConfigError
configmap "sre-debug-config" not found
couldn't find key required-setting
```

## Write the analysis

```text
Symptom:
Pod is not starting.

Mitigation:
Create the missing ConfigMap or remove the env reference.

Root cause:
Deployment references configuration that does not exist in the environment.

Cause category:
Configuration.

Prevention:
Deploy config and workload together.
Add kustomize build validation.
Add server-side dry-run before merge.
Add policy/check that required ConfigMaps exist.
```

## 5 Whys

```text
1. Why is the pod not starting?
   Because container config cannot be created.

2. Why can’t the config be created?
   Because the Deployment references a missing ConfigMap key.

3. Why is the ConfigMap missing?
   Because the app patch introduced a dependency not included in the overlay.

4. Why was that allowed?
   Because there was no validation of config dependencies.

5. What prevents recurrence?
   Bundle config with the workload and add validation checks.
```

## Root-cause fix option

Create:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/03-missing-config-root-cause/configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sre-debug-config
  namespace: sre-debug
data:
  required-setting: "enabled"
```

Then include it as a resource in the overlay:

```yaml
resources:
  - ../../base/sre-debug-app
  - scenarios/lesson-02-symptoms-vs-root-causes/03-missing-config-root-cause/configmap.yaml

patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/03-missing-config-root-cause/missing-config-patch.yaml
```

## Recovery fix

After observing the failure, remove the patch and the ConfigMap resource unless you want to keep it.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix missing config lab"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

---

# 8. Lab 4 — OOMKilled: Resource Limit Symptom-Fix

## Learning Goal

Understand why “bump the memory limit” may be mitigation, not root cause.

## Create the OOM patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/04-resource-limit-symptom-fix/oom-patch.yaml
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
          image: python:3.12-alpine
          command: ["python", "-c"]
          args:
            - |
              x = []
              while True:
                  x.append("x" * 1024 * 1024)
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
```

## Activate

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/04-resource-limit-symptom-fix/oom-patch.yaml
```

## Apply

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce OOMKilled scenario"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage

```bash
kubectl get pods -n sre-debug -o wide
kubectl describe pod -n sre-debug -l app=sre-debug-app
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
kubectl logs -n sre-debug -l app=sre-debug-app --previous --tail=50
```

## Expected evidence

```text
OOMKilled
Exit Code 137
Back-off restarting failed container
```

## Symptom-fix patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/04-resource-limit-symptom-fix/memory-bump-patch.yaml
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
              cpu: 25m
              memory: 128Mi
            limits:
              cpu: 100m
              memory: 256Mi
```

This may delay the failure, but it does not fix the unbounded memory growth.

## Write the analysis

```text
Symptom:
Pod restarts or is OOMKilled.

Mitigation:
Increase memory limit or roll back the bad workload.

Root cause:
The process allocates memory without bound.

Cause category:
Bug, or capacity if memory use is expected and bounded.

Prevention:
Bound memory usage.
Add load/memory testing.
Alert on memory growth before OOM.
Avoid treating memory-limit bumps as final fixes without evidence.
```

## 5 Whys

```text
1. Why did the pod restart?
   Because the container was OOMKilled.

2. Why was it OOMKilled?
   Because memory exceeded the container limit.

3. Why did memory exceed the limit?
   Because the process kept allocating memory.

4. Why did the process keep allocating memory?
   Because the code had unbounded memory growth.

5. What prevents recurrence?
   Fix the code or add bounded memory behavior and tests.
```

## Fix

Remove the OOM patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix OOM lab by removing memory leak patch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

---

# 9. Lab 5 — Service Selector: Layer Mismatch

## Learning Goal

Avoid fixing the wrong layer.

The symptom may look like:

```text
The app is unreachable.
```

But the actual cause is Kubernetes service routing.

## Create the patch

File:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/05-service-selector-layer-mismatch/bad-service-selector.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  selector:
    app: not-the-real-app
```

## Activate

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/05-service-selector-layer-mismatch/bad-service-selector.yaml
```

## Apply

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: introduce service selector mismatch"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

## Triage

```bash
kubectl get pods -n sre-debug --show-labels
kubectl get svc -n sre-debug sre-debug-app -o yaml
kubectl get endpoints -n sre-debug sre-debug-app
kubectl get endpointslice -n sre-debug
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -30
```

Test from inside the cluster:

```bash
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -m 5 http://sre-debug-app
```

## Expected evidence

```text
Pods are Running.
Service exists.
Endpoints are empty.
Curl to Service fails or times out.
```

## Write the analysis

```text
Symptom:
Service request fails.

Mitigation:
Restore the correct selector.

Root cause:
Service selector does not match pod labels.

Cause category:
Configuration.

Layer mismatch:
The symptom looks application-level, but the cause is Kubernetes service discovery/routing.

Prevention:
Add post-deploy smoke test.
Add check that Services have non-empty Endpoints.
Avoid changing labels/selectors without validation.
```

## 5 Whys

```text
1. Why does service access fail?
   Because the Service has no endpoints.

2. Why does it have no endpoints?
   Because the selector does not match pod labels.

3. Why did the selector change?
   Because a Git patch changed it.

4. Why was the change allowed?
   Because there was no smoke test checking Service routing.

5. What prevents recurrence?
   Endpoint validation and in-cluster smoke tests after deploy.
```

## Fix

Remove the patch.

```bash
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Fix service selector lab"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

---

# 10. Lab 6 — Recurring Incident Pattern

## Learning Goal

Train yourself not to call repeat failures “one-offs.”

## Scenario

Run the same service selector break twice.

Create:

```text
apps/test/sre-debug-app/scenarios/lesson-02-symptoms-vs-root-causes/06-recurring-incident-pattern/recurring-service-selector.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sre-debug-app
  namespace: sre-debug
spec:
  selector:
    app: recurring-wrong-label
```

## Round 1

Activate the patch:

```yaml
patches:
  - path: scenarios/lesson-02-symptoms-vs-root-causes/06-recurring-incident-pattern/recurring-service-selector.yaml
```

Commit:

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: recurring incident round 1"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

Observe:

```bash
kubectl get pods -n sre-debug --show-labels
kubectl get endpoints -n sre-debug sre-debug-app
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -m 5 http://sre-debug-app
```

Fix by removing the patch. Commit and reconcile.

## Round 2

Re-add the same patch.

Commit:

```bash
git add apps/test/sre-debug-app
git commit -m "Lab 02: recurring incident round 2"
git push
flux reconcile kustomization test-apps -n flux-system --with-source
```

Observe the same failure.

## Write the recurrence note

```markdown
# Recurrence Analysis

## Incident 1

Symptom:

Mitigation:

Root cause:

## Incident 2

Symptom:

Mitigation:

Root cause:

## Pattern

What repeated?

## Trigger

What change introduced it?

## Prevention

What control would stop this class of failure?

## Action Item

Owner:
Deadline:
Verification:
```

## Correct lesson

If the same class of failure happens twice, it needs a guardrail.

Examples:

```text
Service endpoint validation
In-cluster smoke test
Policy check on Service selectors
PR checklist for label/selector changes
```

---

# 11. Lab 7 — Postmortem Drill

## Learning Goal

Practice writing a useful postmortem that captures symptom, mitigation, root cause, and prevention.

Create:

```text
journal/debugging/lesson-02-postmortem-template.md
```

```markdown
# Postmortem: <Incident Title>

**Date**:
**Severity**:
**Duration**:
**Author**:

## Summary

What happened?
Who or what was impacted?
How long did it last?

## Symptom

What alerted or appeared broken?

## Mitigation

What stopped the immediate impact?

## Timeline

- HH:MM - Change merged
- HH:MM - Flux reconciled
- HH:MM - Symptom observed
- HH:MM - Events checked
- HH:MM - Layer identified
- HH:MM - Mitigation applied
- HH:MM - Recovery confirmed
- HH:MM - Root cause identified

## Root Cause

What underlying condition caused the symptom?

## 5 Whys

1. Why?
2. Why?
3. Why?
4. Why?
5. Why?

## Cause Category

Choose one:

- Bug
- Configuration
- Capacity
- Process
- Architecture

## What Worked

## What Did Not Work

## Action Items

- [ ] [Owner / Due Date] Specific action
- [ ] [Owner / Due Date] Verification action

## Recurrence Prevention

How will we know this cannot happen the same way again?
```

## Grading rule

A weak postmortem says:

```text
Restarted pod. Resolved.
```

A strong postmortem says:

```text
Pod restarted because the container exited due to a bad command override introduced in commit X. The fix was to remove the command override. Prevention is a review check and a smoke test that catches startup failure before the change is considered complete.
```

---

# 12. Suggested Practice Order

Run the labs in this order:

```text
1. CrashLoopBackOff: Symptom vs Root Cause
2. ImagePullBackOff: Mitigation vs Real Fix
3. Missing Config: Configuration Root Cause
4. OOMKilled: Resource Limit Symptom-Fix
5. Service Selector: Layer Mismatch
6. Recurring Incident Pattern
7. Postmortem Drill
```

Do not run multiple failure patches at the same time.

---

# 13. Quick Reset Checklist

After every lab:

```bash
# Remove active patch from kustomization.yaml
git add apps/test/sre-debug-app/kustomization.yaml
git commit -m "Reset sre-debug-app to healthy state"
git push

flux reconcile kustomization test-apps -n flux-system --with-source

kubectl get pods -n sre-debug -o wide
kubectl get events -n sre-debug --sort-by=.lastTimestamp | tail -20
```

Healthy state:

```text
Pods are 1/1 Running.
No new warning events.
Service has endpoints.
test-apps is Ready=True.
```

Check endpoints:

```bash
kubectl get endpoints -n sre-debug sre-debug-app
```

Check service access:

```bash
kubectl run curl-test -n sre-debug --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -m 5 http://sre-debug-app
```

---

# 14. Final Mastery Questions

After finishing the labs, answer these in your own words:

```text
1. What is the difference between a symptom and a root cause?

2. Why is restarting a pod usually mitigation, not a fix?

3. How can increasing memory limits hide the real problem?

4. What evidence tells you an incident is recurring?

5. What makes an action item strong instead of vague?

6. How does GitOps help root cause analysis?

7. How can GitOps also make incidents worse if validation is weak?
```

When you can answer these clearly using your lab evidence, you have solidified Lesson 2.
