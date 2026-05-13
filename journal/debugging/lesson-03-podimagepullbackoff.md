# Lesson 01 Incident Note: Pod Image Pull Backoff

## Date

May 13th 2026

## Scenario

Kubernets cannot pull the image, and as a result the application never starts.

This Lab will examine `ImagePullBackOff` which is a pod lifecycle failure before the application even runs
The main cause is that the image does not exist, or the image name is misspelled and therefore a misspelled image name does not exist.

 I will create a patch to the base kustomization of the application that will cause the failure. Push it to `git` and reconcile wiht `flux`


## Symptom

Flux would produce an exceeded deadline result
the pod was not created because the image did not exist
Did a two minute triage
1. check pods
2. check events
3. check kustomization

Dug deeper and check pod description

## Scope

narrowed down the scope to the pod layer
container image could not be pulled, and as a result the application did not start


## Events

Command:

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30