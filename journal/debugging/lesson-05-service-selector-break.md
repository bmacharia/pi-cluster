# Lesson 01 Incident Note: <Scenario Name>

## Date
May 13th 2025

## Scenario

What happens when the pods are healthy, the service exists, but the service selector no longer matches the pod labels, and traffic does not route to the pods. What it is being demonstrated is that healthy pods do not mean a healthy service

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