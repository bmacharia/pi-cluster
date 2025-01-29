# Secrets Management with SOPS

This document provides a technical overview of managing Kubernetes secrets using [SOPS](https://github.com/mozilla/sops) and [age](https://github.com/FiloSottile/age). By following these steps, you can securely encrypt and store sensitive configuration data in a Kubernetes cluster.

---

## Installing Dependencies

To install both **sops** and **age** using Homebrew:

```bash
brew install sops age

age-keygen -o age.agekey

cat age.agekey

```

## Generating an age Key Pair

age-keygen -o age.agekey

Inspsct the generatee key:

`cat age.agekey`

## Encrypting a Kubernetes Secret

#### Create a YAML manifest for a Kubernetes secret using `kubeclt`

```bash
kubectl create secret generic test-secret \
--from-literal=user=admin \
--from-literal=password=mischa \
--dry-run=client \
-o yaml > test-secret.yaml
```

#### Export the age public key as an enviromnet variable:

```bash
export AGE_PUBLIC=age1zd5n9zx0dsdwdggjuvz32ppngn45gk0tcnwlwfs53ev7szefmdfqarudsl
```

#### Encrypt the secret with SOPS:

```bash
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml
```

#### Verify the encrypted contents

```bash
cat test-secret.yaml
```

#### Adding the Private Key to the Cluster

To decrypt secrets on the cluster side `(e.g.,via Flux)`, you must add the private key as a Kubernetes secret:

```bash
cat age.agekey | kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

## Congfiguring SOPS in the Respository

Withing your respository `(e.g., in the clusters/staging)` directory create or update the `.sops.yaml` configuration file. This instructs SOPS to encrypt/decrypt matching files automatically:

```bash
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1zd5n9zx0dsdwdggjuvz32ppngn45gk0tcnwlwfs53ev7szefmdfqarudsl
```

## Configure Flux Kustomization

In your Flux Kustomization file (for example, apps.yaml), add the following under spec:

```bash
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

Make sure this is at the same indentation level as other top-level keys under spec(e.g.,prune:true)

## Managing CloudFlare Secrets

To manafe Cloudflare tunnel credentials (as an example)

```bash
kubectl create secret generic tunnel-credentials \
--from-file=credentials.json=59c97568-9eda-4dbf-9857-b6cf9cf91259.json \
--dry-run=client \
-o yaml > cloudflare-secret.yaml
```

Commit this file to your respository and allow Flux CD to handle the deployment updates and details
