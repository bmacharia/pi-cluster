# Kubernetes Distribution Options for Home Lab

Kubernetes is a collection of binaries that operate on a Linux distribution. There are several options available for setting up a Kubernetes cluster.

A Kubernetes distribution is a packaged version of Kubernetes that:

- Bundles the core Kubernetes components
- Provides its own installation and management methods
- May modify or optimize certain components
- Often includes additional tools and features

## Kubeadm

- Utilized in CKA certification
- Suitable for deep learning but challenging for long-term management
- Requires manual management of certificates and CNI

## MicroK8S

- Developed by Canonical
- Easy to install but follows an opinionated approach
- Includes bundled add-ons with limited configuration options

## Talos Linux

- Production-grade, secure, and hardened Kubernetes
- Immutable OS that takes over the entire disk
- No SSH access, only API communication
- Considered more advanced and abstracts away installation details

## K3S (Chosen for this Course)

- Balances ease of use with learning opportunities
- Single binary installation on existing Linux systems
- Allows for OS-level troubleshooting and exploration
- Optimized for ARM (ideal for Raspberry Pi)
- Lightweight and easy to install (one command)
- Powers Rancher, relevant for enterprise Kubernetes management
- Easy to upgrade and add nodes
- Flexible for networking plugins and customization

Future courses will cover more advanced options like Talos Linux. The GitOps approach will facilitate easy transfer of setups between distributions.
