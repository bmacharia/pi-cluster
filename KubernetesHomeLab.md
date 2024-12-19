# Kubernetes HomeLab

The hardware I will be using for my homelab intially will be a Raspberry PI 5 8GB version. I will add more Raspberry Pi Nodes to the Kubernetes Cluster as Time goes on. For now I will operating a single Node Kubernetes Cluster.

## Kubernetes Distrubution Options for Home Lab

Kubernetes is a set of binaries running on a Linux distribution.

Several options exist for setting up a Kubernetes cluster.

## A Kubernetes distribution is a packaged version of Kubernetes that

- Bundles the core Kubernetes components
- Provides its own installation and management methods
- May modify or optimize certain components
- Often includes additional tools/features

- Kubeadm:

  - Used in CKA certification

  - Good for deep learning but challenging for long-term management

  - Requires manual certificate and CNI management

- MicroK8S:

  - Developed by Canonical

  - Easy to install but has an opinionated approach

  - Bundled add-ons with limited configuration options

Talos Linux:

- Production-grade, secure, and hardened Kubernetes

- Immutable OS that takes over entire disk

- No SSH access, only API communication

- Considered more advanced and abstracts away installation details

- K3S (chosen for this course):

  - Strikes a balance between ease of use and learning opportunities

  - Single binary installation on existing Linux systems

  - Allows for OS-level troubleshooting and exploration

  - Optimized for ARM (good for Raspberry Pi)

  - Lightweight and easy to install (one command)

  - Powers Rancher, relevant for enterprise Kubernetes management

  - Easy to upgrade and add nodes

  - Flexible for networking plugins and customization

  In the future I will cover more advanced options like Talos Linux

  GitOps approach will make it easy to transfer setups between distributions

# Installing K3s on Raspberry Pi

- Update and upgrade systems packages
- Install VIM
- Enable cgroup memory parameters in boot configurtion
- Reboot Raspberry Pi

```
sudo vim /boot/firmware/cmdline.txt

# add:

# Make sure that it is added to the existing line.

# Not on a new line

cgroup_memory=1 cgroup_enable=memory

sudo reboot

# Check if it worked

cat /proc/cmdline
```

Run the K3s installation script with the helm controller disabled

```
sudo su -

curl -sFL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=helm-controller" sh

sudo systemctl status k3s.service
```

Configure remote access:

- Copy K3s YAML file from Raspberry file to local machince
- Create .kube directory in home folder
- Move and remane K3s YAML to .kube/config
- Update server IP address in the config file

```
cd

sudo cp /etc/rancher/k3s/k3s.yaml .

exit

# On main system

scp mischa@192.168.10.60:/home/mischa/k3s.yaml .

vim k3s.yaml # edit the IP

mkdir ~/.kube

mv k3s.yaml .kube/config



```

Verifying installation:

- Use kubectl commands to interact with the cluster

- Check running pods and namespaces

• Understanding K3s architecture:

- K3s runs as a single binary managed by systemd

- Uses containerd as the container runtime

- Kubernetes pods are collections of containers

• Next steps:

- Set up Flux and GitOps for managing the cluster

# GitOps Theory and Definition

• GitOps Definition:

- A set of best practices where the entire code delivery process is controlled via Git

- Includes infrastructure and application definition as code

- Uses automation for updates and rollbacks

• Key GitOps Principles:

- Entire system (infrastructure and applications) is described declaratively

- Desired system state is versioned in Git

- Approved changes are automatically applied to the system

- Software agents ensure correctness and alert on divergence

• GitOps Workflow:

- Git repository holds all configuration for applications and infrastructure

- Cluster pulls changes and deployment information

- GitOps controller runs a constant loop to match Git state with cluster state

• Benefits of GitOps:

- Provides version control and history of all changes

- Enables collaboration through Git workflows

- Improves security by reducing direct cluster access

- Facilitates automation for updates and rollbacks

• Flux vs. Argo CD:

- Flux chosen for this course due to:

  • Simpler setup and less overwhelming interface

  • Encourages CLI usage and GitOps thinking

  • Native integration with Azure Kubernetes Service (AKS)

  • Heavy reliance on Kustomize for manifest management

• Kustomize:

- Templating language for Kubernetes manifests

- Allows for declarative management of Kubernetes objects

- Important skill for Kubernetes engineers

• Reasons for Using GitOps in Home Lab:

- Provides practical experience with industry-standard practices

- Creates a public record of learning and skills

- Facilitates collaboration and version control

# Installing Flux

## Setting Up GitOps with Flux on a Kubernetes Cluster

- I will set up a GitOps workflow using Flux on a fresh Kubernetes cluster

- A new GitHub repository is created for the project

- Optional: DevContainer configuration

## Installing Flux

- Flux needs to be installed on the cluster to facilitate the GitOps reconciliation loop

- Prerequisites: Kubernetes cluster and GitHub personal access token

### Creating a GitHub Personal Access Token

- Go to GitHub Settings > Developer Settings > Personal Access Tokens

- Generate a new classic token with 'repo' permissions

- Store the token in an environment variable: export GITHUB_TOKEN=<your-token>

- Also export your GitHub username: export GITHUB_USER=<your-username>

### Installing Flux CLI

- Various installation methods available (e.g., Homebrew, curl)

- Verify installation with which flux

### Checking Cluster Compatibility

- Run flux check --pre to ensure the cluster meets Flux requirements

### Bootstrapping Flux on the Cluster

- Use the command:

```
flux bootstrap github \

  --owner=$GITHUB_USER \

  --repository=pi-cluster \

  --branch=main \

  --path=./clusters/staging \

  --personal
```

- This installs Flux controllers and commits manifests to the specified repository

## Results

- Flux creates necessary manifests in the GitHub repository

- New namespace 'flux-system' is created on the cluster with Flux components

- GitOps controller is now active on the cluster

## Next Steps

- Ready to deploy applications using GitOps methodologies

# GitOps Deployment

I will deploy the application (**LinkDing**, a bookmark manager) to a Kubernetes cluster using Flux and GitOps principles.

• I will follow Flux's recommended repository structure, organizing our files into separate directories for apps, base configurations, and environment-specific configurations (staging, production).

• I will create several YAML files:

- **apps.yaml**: This tells Flux to look at the apps/staging directory

- **kustomization.yaml** files: Here we'll define resources and configurations for the application

- **namespace.yaml**: We'll create a dedicated namespace for the application

- **deployment.yaml**: We'll define the Kubernetes deployment for the application

• **The path Flux follows to deploy a application:**

1. Flux reads the clusters/staging directory

2. Finds and applies our apps.yaml file

3. Looks at the apps/staging directory

4. Finds our Linkding kustomization file

5. Applies the resources we defined in the base configuration

• After a commit and push our changes, Flux automatically detects and applies the new configuration without any manual intervention needed.

• I will verify our deployment by checking the namespace, pods, and using port-forwarding to access the application.

• What this approach gives is a version-controlled, declarative management of Kubernetes resources, and we'll see firsthand the benefits of GitOps.

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
  name: linkding



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
      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090



kubectl port-forward pod/linkding 8080:9090
```

# Adding Storage

Adding persistent storage to a LinkedIn deployment in Kubernetes

• Creating a persistent volume claim (PVC) using GitOps

• Using Flux to manage Kubernetes resources

• Demonstrating data persistence across pod restarts

• Exploring security risks of running containers as root

• Introduction to K9S for cluster management

• Importance of securing applications in Kubernetes

1. Persistent storage in Kubernetes:

   • Creating a PVC

   • Adding the PVC to the deployment configuration

   • Demonstrating data persistence when pods are deleted and recreated

2. GitOps workflow:

   • Creating and modifying YAML files for Kubernetes resources

   • Using Flux to detect and apply changes from Git repository

   • Importance of adding new resources to the Flux customization file

3. Kubernetes commands and tools:

   Using kubectl for various operations (describe, exec, port-forward)

   • Introduction to K9S as an alternative to kubectl

   • Using kubens (aliased as 'kn') for namespace management

4. Security concerns:

   • Demonstrating the risks of running containers as root

   • Ability to install packages and modify container contents

   • Introduction to the need for securing applications in Kubernetes

5. Linkding application specifics:

   • Creating a superuser for the Linkding application

   • Demonstrating user persistence with added storage

   • Using port-forwarding to access the application locally

```
k exec -it linkding-7bffb6cdb9-h7mnm -- python manage.py createsuperuser --username=mischa --email=mischa@example.com
```

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: linkding-data-pvc
spec:
accessModes: - ReadWriteOnce
resources:
requests:
storage: 1Gi

---

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
containers: - name: linkding
image: sissbruecker/linkding:1.31.0
ports: - containerPort: 9090

          volumeMounts:
            - name: linkding-data
              mountPath: /etc/linkding/data
      volumes:
        - name: linkding-data
          persistentVolumeClaim:
            claimName: linkding-data-pvc

```

```

# Enhancing Application Security

Now that the application linkding is deployed to the Kubernetes cluster and Persistent Volume Storage has been added to the Pod the the Application is running on, It is of utmost importantce to improve the security of the application before exposing it to the public Internet

The main focus in to modify the deployment configuration to enhance security.

The Key changes to the deployment are as followed

- Adding a Security Context
- Spefifying an FS group for volume permisions
- Setting run as user and run as group

To determine the appropriate user, I will exammine the /etc/passwd file in the container

the www-data-user (ID 33) is identifgied as the intended user for the application

### Security Context Modifications

- Set the container to run as user 33
- Update volume permissions to user 33
- Disable privilege escalation

I will commit and push these changes to the repository.
Flux will be used to reconcile and apply the configuration
The updated POD will run with restricted permissions

The restricted permisssions incluse

- Unable to perform packkage updates of installations
- Cannot escalate to root user
- Limited to pre-installed functionality

I will verify that the application still functions correctly with the new security settings. This security enhancements prepares the application to be exposed to the public internet

Looking Ahead I will be exposing the application to the internet using Cloudflare tunnels
