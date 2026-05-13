# Lesson 01 Incident Note: <Scenario Name>

## Date
May 13th 2026

## Symptom

sre-debug app pods entered CrashLoopbackOff

## Scope
 Only sre-debug-app in sre0debug namespace


## Events

the container failed and kubelet back off restarting it.
The failure was at the pod level
Root Cause: container process exitec immediatley
Fix: In the kustomization file I removed the patch, pushed to git, and reconciled it with Flux