# Enhancing Application Security

## Overview

This documentation outlines the steps to improve the security of an application prior to exposing it to the public internet. The primary focus is on modifying the deployment configuration to enhance security measures.

## Key Security Enhancements

### Deployment Configuration Changes

1. **Add a Security Context**
2. **Specify an FS Group for Volume Permissions**
3. **Set `runAsUser` and `runAsGroup`**

### Determining the Appropriate User

To identify the correct user for the application, inspect the `/etc/passwd` file within the container. The `www-data` user (UID 33) is selected as the intended user for this application.

### Security Context Modifications

- Set containers to run as user `33` (www-data).
- Update volume permissions to user `33`.
- Disable privilege escalation.

## Implementation Steps

### Commit and Push Changes

- Commit the updated configuration to the repository.
- Use Flux to reconcile and apply the new configuration changes.

### Restricted Pod Permissions

The updated pod will operate with the following security restrictions:

- **Package Updates or Installations**: Disabled
- **Root Escalation**: Prevented
- **Functionality**: Limited to pre-installed capabilities

### Verification

- Ensure the application functions correctly under the new security settings.
- Confirm that the security enhancements are effective and that the application is ready for public exposure.

## Next Steps

The next module will focus on exposing the application to the public internet using Cloudflare Tunnels.

## Example Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      securityContext:
        fsGroup: 33 # www-data group ID
        runAsUser: 33 # www-data user ID
        runAsGroup: 33 # www-data group ID

      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090

          securityContext:
            allowPrivilegeEscalation: false

          volumeMounts:
            - name: linkding-data
              mountPath: /etc/linkding/data
      volumes:
        - name: linkding-data
          persistentVolumeClaim:
            claimName: linkding-data-pvc
```
