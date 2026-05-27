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