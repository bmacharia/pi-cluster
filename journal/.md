# Setting Up Renovate in Kubernetes:

This a step-by-step guide to configuring Renovate, a dependency update tool, within a Kubernetes cluster. We'll create secrets, define a CronJob, set up a ConfigMap, and use Kustomize for resource management. By the end, you'll have Renovate running hourly to manage dependencies in your GitHub repository.

## Prerequisites

- A Kubernetes cluster with `kubectl` configured.
- [SOPS](https://github.com/mozilla/sops) installed for secret encryption.
- An AGE public key (`$AGE_PUBLIC`) for encryption.
- A GitHub Personal Access Token (PAT) for Renovate.

## Step 1: Creating and Encrypting the Renovate Secret

First, we'll create a Kubernetes secret containing the Renovate token and encrypt it using SOPS for security.

### Generate the Secret

Run the following command to create a secret manifest with your GitHub PAT:

```bash
kubectl create secret generic renovate-container-env \
  --from-literal=RENOVATE_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx \
  --dry-run=client \
  -o yaml > renovate-container-env.yaml
```

This command:

- Creates a secret named `renovate-container-env`.
- Stores the `RENOVATE_TOKEN` as a literal value.
- Outputs the result in YAML format to `renovate-container-env.yaml`.

### Encrypt the Secret with SOPS

Encrypt the sensitive data in the secret using SOPS and an AGE public key:

```bash
sops --age=$AGE_PUBLIC \
  --encrypt \
  --encrypted-regex '^(data|stringData)$' \
  --in-place renovate-container-env.yaml
```

- `--age=$AGE_PUBLIC`: Specifies the AGE public key for encryption.
- `--encrypted-regex '^(data|stringData)$'`: Encrypts only the `data` and `stringData` fields.
- `--in-place`: Modifies the file directly.

The resulting `renovate-container-env.yaml` is now encrypted and secure.

## Step 2: Defining the Renovate Namespace

Create a dedicated namespace for Renovate resources:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: renovate
```

This YAML defines a namespace called `renovate` to isolate Renovate-related resources.

## Step 3: Configuring the Renovate CronJob

Next, define a Kubernetes CronJob to run Renovate hourly.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renovate
  namespace: renovate
spec:
  schedule: "@hourly"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: renovate
              image: renovate/renovate:latest
              args:
                - airmanbabu/pi-cluster
              envFrom:
                - secretRef:
                    name: renovate-container-env
                - configMapRef:
                    name: renovate-configmap
          restartPolicy: Never
```

Key configurations:

- `schedule: "@hourly"`: Runs the job every hour.
- `concurrencyPolicy: Forbid`: Prevents overlapping runs.
- `image: renovate/renovate:latest`: Uses the latest Renovate image.
- `args`: Specifies the GitHub repository (`airmanbabu/pi-cluster`) to manage.
- `envFrom`: Loads environment variables from the encrypted secret and a ConfigMap.

## Step 4: Creating the Renovate ConfigMap

Define a ConfigMap to store Renovate's configuration settings:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: renovate-configmap
  namespace: renovate
data:
  RENOVATE_AUTODISCOVER: "false"
  RENOVATE_GIT_AUTHOR: "Renovate Bot <bot@renovateapp.com>"
  RENOVATE_PLATFORM: "github"
```

- `RENOVATE_AUTODISCOVER: "false"`: Disables automatic repository discovery.
- `RENOVATE_GIT_AUTHOR`: Sets the commit author for Renovate updates.
- `RENOVATE_PLATFORM: "github"`: Specifies GitHub as the platform.

## Step 5: Managing Resources with Kustomize

Use Kustomize to manage the Kubernetes resources efficiently:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - secrets.yaml
  - configmap.yaml
  - cronjob.yaml
```

- Lists the resource files (`secrets.yaml`, `configmap.yaml`, `cronjob.yaml`) to be applied together.
- Run `kubectl apply -k .` in the directory containing this `kustomization.yaml` to deploy.

## Step 6: Configuring Renovate for Kubernetes Files

In the root of your repository (`mischavandenburg/pi-cluster`), add a `renovate.json` file to configure Renovate's behavior:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "kubernetes": {
    "fileMatch": ["\\.yaml$"]
  }
}
```

- `"$schema"`: References the Renovate configuration schema.
- `"kubernetes"."fileMatch"`: Instructs Renovate to process `.yaml` files for dependency updates.

## Deployment

1. Save the YAML files in a directory (e.g., `renovate-k8s/`).
2. Ensure `renovate.json` is in the root of your GitHub repository.
3. Apply the resources:
   ```bash
   kubectl apply -k renovate-k8s/
   ```
4. Verify the CronJob is scheduled:
   ```bash
   kubectl get cronjob -n renovate
   ```

Renovate will now run hourly, updating dependencies in your Kubernetes manifests and creating pull requests on GitHub.

## Conclusion

Renovate is now in Kubernetes with encrypted secrets, a scheduled CronJob, and Kustomize for resource management. This setup ensures automated dependency updates while maintaining security and scalability.
