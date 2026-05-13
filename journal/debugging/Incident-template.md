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