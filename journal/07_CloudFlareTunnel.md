# Cloudflare Tunnel Deployment for Kubernetes

## References

- [Cloudflare Kubernetes Deployment Guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deploy-tunnels/deployment-guides/kubernetes/)
- [Cloudflare Tunnel CLI Guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/)

## Disadvantage

A notable downside of this setup is the requirement to include a token in the code, which is not ideal for security.

## Configuring Tunnel via CLI

### Installing CloudflareD

1. Run the following command to log in and generate a certificate:

   ```bash
   cloudflared tunnel login
   ```

   This creates a certificate at:

   ```
   /home/devopbiddy/.cloudflared/cert.pem
   ```

2. Create a new tunnel:

   ```bash
   cloudflared tunnel create ldpi
   ```

3. Store tunnel credentials in a Kubernetes secret:
   ```bash
   kubectl create secret generic tunnel-credentials \
   --from-file=credentials.json=59c97568-9eda-4dbf-9857-b6cf9cf91259.json
   ```

## Adding to Cloudflare DNS

1. Go to Cloudflare DNS and add a new record.
2. Select `CNAME` and enter the following details:
   ```
   59c97568-9eda-4dbf-9857-b6cf9cf91259.cfargotunnel.com
   ```

## Adding Kubernetes Manifests

### Service Definition

The following YAML defines a service for the application:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: linkding
spec:
  ports:
    - port: 9090
  selector:
    app: linkding
  type: ClusterIP
```

### Cloudflare Tunnel Deployment

The YAML below configures the Cloudflare tunnel:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 2 # You could also consider elastic scaling for this deployment
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config/config.yaml
            - run
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config
              readOnly: true
            - name: creds
              mountPath: /etc/cloudflared/creds
              readOnly: true
      volumes:
        - name: creds
          secret:
            secretName: tunnel-credentials
        - name: config
          configMap:
            name: cloudflared
            items:
              - key: config.yaml
                path: config.yaml
```

### ConfigMap for Tunnel Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared
data:
  config.yaml: |
    tunnel: ldpi
    credentials-file: /etc/cloudflared/creds/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true

    ingress:
    - hostname: ldpi.airmanbabu.net
      service: http://linkding:9090
    - hostname: hello.example.com
      service: hello_world
    - service: http_status:404
```
