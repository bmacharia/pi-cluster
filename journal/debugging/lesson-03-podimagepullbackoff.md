# Lesson 01 Incident Note: Pod Image Pull Backoff

## Date

May 13th 2026

## Scenario

Kubernets cannot pull the image, and as a result the application never starts.

This Lab will examine `ImagePullBackOff` which is a pod lifecycle failure before the application even runs
The main cause is that the image does not exist, or the image name is misspelled and therefore a misspelled image name does not exist.

 I will create a patch to the base kustomization of the application that will cause the failure. Push it to `git` and reconcile wiht `flux`


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