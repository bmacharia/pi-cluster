# Deploying Audiobookshelf on Kubernetes

## Initial Setup

The deployment begins by containerizing Audiobookshelf to operate on port `3005`. To manage this port flexibly, a Kubernetes `ConfigMap` is implemented, enabling environment-driven configuration. A `ClusterIP` service exposes Audiobookshelf internally on port `3005`, ensuring secure access restricted to the cluster network.

For manual testing and initial setup, Kubernetes' `port-forward` feature is employed:

```bash
kubectl -n audiobookshelf port-forward svc/audiobookshelf 8080:3005
```

This allows local accessibility, facilitating preliminary testing like uploading audio files directly to the service.

## Persistent Storage Integration

In stage two, durability becomes essential. Following the Audiobookshelf documentation, three persistent volume claims (PVCs) are defined:

- `audiobookshelf-config`
- `audiobookshelf-metadata`
- `audiobookshelf-audiobooks`

Each PVC requests 1Gi of storage, ensuring dedicated, persistent storage for configuration, metadata, and audiobook files, respectively.

The deployment manifests are updated to integrate these PVCs, mounting them explicitly within the container at specified paths:

```yaml
volumeMounts:
  - mountPath: /config
    name: audiobookshelf-config
  - mountPath: /metadata
    name: audiobookshelf-metadata
  - mountPath: /audiobooks
    name: audiobookshelf-audiobooks
```

## Enhancing Security and Public Accessibility

Security considerations are paramount in stage three. Audiobookshelf is configured to run as the non-root user `node` by setting Kubernetes security contexts:

```yaml
securityContext:
  fsGroup: 1000
  runAsUser: 1000
  runAsGroup: 1000
```

Public accessibility is facilitated through Cloudflare Tunnels (`cloudflared`), avoiding direct exposure to the internet. The tunnel credential file is securely managed using Mozilla SOPS for encryption and Flux for deployment automation:

```bash
cloudflared tunnel create audiobooks
kubectl create secret generic tunnel-credentials \
  --from-file=credentials.json=<UUID>.json \
  --dry-run=client -o yaml > cloudflare-secret.yaml

sops --age=$AGE_PUBLIC \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place cloudflare-secret.yaml
```

DNS is then configured to point the tunnel subdomain (e.g., `audiobooks.airmanbabu.net`) to Cloudflare, completing the ingress setup.

### Cloudflared Deployment and ConfigMap

To maintain high availability, the Cloudflared deployment uses multiple replicas (`replicas: 2`) with Kubernetes liveness probes ensuring consistent connectivity to Cloudflare's edge network.

The `cloudflared` deployment reads configuration dynamically from a Kubernetes `ConfigMap`, enabling easier maintenance and templating:

```yaml
config.yaml: |
  tunnel: audiobooks
  credentials-file: /etc/cloudflared/creds/credentials.json
  metrics: 0.0.0.0:2000
  no-autoupdate: true

  ingress:
  - hostname: audiobooks.mischavandenburg.net
    service: http://audiobookshelf:3005
  - hostname: hello.example.com
    service: hello_world
  - service: http_status:404
```

## Conclusion

This Kubernetes-based Audiobookshelf deployment demonstrates a robust blend of application portability, configuration-driven infrastructure, security best practices, and accessibility. Each stage incrementally refines the setup, emphasizing secure and scalable architecture while integrating modern tools like SOPS, Flux, and Cloudflare Tunnels.

The journey from basic deployment to secure public access underscores Kubernetes' flexibility and the importance of iterative refinement in modern DevOps practices.
