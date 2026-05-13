# Lesson 01 Incident Note: Scheduling Pending Pod

## Date
May 13th 2026


## Scenario

The pod running on the cluster requests for a large amount of compute resources that exceed the physical reality of the node. In other words that amount of cpu does not exists and the node cannit run the pod, and the pod stays in a pending state, until appropriate resources are available. It is a schduleing issue and not neccessairly a pod issue. This is a node layer issue.

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