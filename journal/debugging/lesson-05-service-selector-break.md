# Lesson 01 Incident Note: <Scenario Name>

## Date
May 13th 2025

## Scenario

What happens when the pods are healthy, the service exists, but the service selector no longer matches the pod labels, and traffic does not route to the pods. What it is being demonstrated is that healthy pods do not mean a healthy service. 

When it comes to services in kubernetes, what that means is that it has to do with networking, communcatin between pods and pods on other nodes. This is a cluster networkinf issue

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