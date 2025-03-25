## Deploying an Ingress for Linkding

### Overview

- **Linkding** is a simple bookmarking service that you can self-host.
- **Ingress** is used to expose HTTP and HTTPS routes from outside the Kubernetes cluster to services within the cluster.
- **Traefik** is configured here as the Ingress Controller via the `ingressClassName`.
- Steps to make an `/etc/hosts` entry are shown to resolve your custom domain (`lds.airmanbabu.net`) to the IP address of your Kubernetes node or LoadBalancer.
- You can verify DNS resolution with `nslookup`.

### Ingress Configuration

Below is the YAML configuration for the **Ingress** resource:

<details>
<summary><strong>Ingress YAML</strong></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linkding
  namespace: linkding
spec:
  ingressClassName: traefik
  rules:
    - host: lds.airmanbabu.net
      http:
        paths:
          - backend:
              service:
                name: linkding
                port:
                  number: 9090
            path: /
            pathType: Prefix
```

</details>

#### Explanation of Key Fields

- **`apiVersion` and `kind`**: Using the `networking.k8s.io/v1` API for an `Ingress` object.
- **`metadata.namespace`**: Namespace where the Linkding application and this Ingress resource reside (`linkding`).
- **`ingressClassName`**: Specifies that we’re using `traefik` as the Ingress Controller.
- **`rules.host`**: The domain name (host) for which this Ingress routes requests. In this example, `lds.airmanbabunet`.
- **`paths.backend.service.name`**: The Kubernetes Service name serving Linkding.
- **`paths.backend.service.port.number`**: The port on which Linkding is exposed inside the cluster (`9090`).
- **`pathType`**: Defines how the path should be interpreted. `Prefix` matches all requests that begin with `/`.

### Local Host Resolution

If you’re testing locally or on a network without a DNS record pointing to your Ingress, you can modify the `/etc/hosts` file to resolve the domain:

```bash
sudo vim /etc/hosts
```

Add the following line (adjust the IP address if needed):

```
192.168.10.60 lds.airmanbabu.net
```

> **Note**: When you set this host entry, your local machine will resolve `lds.airmanbabu.net` to the IP address `192.168.10.60`.

### DNS Verification

You can verify that DNS resolution is working as expected:

```bash
nslookup lds.airmanbabu.net
```

- Replace with the host you actually configured (`lds.airmanbabu.net`) if you want to check that record specifically, or consult your DNS provider or local host entries to ensure it resolves to the expected IP address.

### Summary

1. **Ingress Setup**: Deployed an Ingress resource with Traefik for the Linkding service.
2. **Local DNS Override**: Configured your system’s `/etc/hosts` for testing or development environments.
3. **Validation**: Used `nslookup` to confirm that DNS records point correctly, ensuring your domain routes to the Ingress Controller.

By following these steps, you’ll have a functional, accessible Linkding service exposed at the domain you’ve specified. This method is useful for local development and testing when you don’t have a public DNS entry. Once ready for production, switch to a proper DNS provider and point your domain’s `A` record to your Kubernetes Ingress IP or LoadBalancer.

---
