# Installing and Configuring K3s on a Raspberry Pi

## **System Preparation**

### **Update and Upgrade System Packages**

Ensure the Raspberry Pi is running the latest updates:

```bash
sudo apt update && sudo apt upgrade -y
```

### **Install Vim**

Install the Vim editor for configuration editing:

```bash
sudo apt install vim -y
```

### **Enable cgroup Memory Parameters**

Modify the boot configuration to enable cgroup memory parameters:

1. Open the boot configuration file:
   ```bash
   sudo vim /boot/firmware/cmdline.txt
   ```
2. Append the following parameters to the **existing line** (do not create a new line):
   ```
   cgroup_memory=1 cgroup_enable=memory
   ```
3. Save the file and exit the editor.

### **Reboot the System**

Reboot the Raspberry Pi to apply the changes:

```bash
sudo reboot
```

### **Verify cgroup Configuration**

Ensure the cgroup parameters are correctly applied:

```bash
cat /proc/cmdline
```

Confirm the output includes:

```plaintext
cgroup_memory=1 cgroup_enable=memory
```

## **Installing K3s**

### **Run K3s Installation Script**

Install K3s with the Helm controller disabled:

```bash
sudo su -
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=helm-controller" sh
```

### **Verify K3s Service**

Check the status of the K3s service to ensure it is running:

```bash
sudo systemctl status k3s.service
```

## **Configuring Remote Access**

### **Copy K3s Configuration File**

Transfer the K3s configuration file from the Raspberry Pi to your local machine:

1. On the Raspberry Pi:
   ```bash
   cd
   sudo cp /etc/rancher/k3s/k3s.yaml .
   exit
   ```
2. Copy the file to your local system:
   ```bash
   scp <username>@<Raspberry_Pi_IP>:/home/<username>/k3s.yaml .
   ```

### **Configure kubectl on Local Machine**

1. Edit the `k3s.yaml` file to update the `server` field with the Raspberry Pi's IP address:
   ```bash
   vim k3s.yaml
   ```
2. Move and rename the file to the Kubernetes configuration directory:
   ```bash
   mkdir -p ~/.kube
   mv k3s.yaml ~/.kube/config
   ```

## **Verifying Installation**

### **Interact with the Cluster**

Use `kubectl` to interact with the K3s cluster:

```bash
kubectl get nodes
```

### **Check Running Pods and Namespaces**

List all pods and namespaces to verify the cluster's functionality:

```bash
kubectl get pods --all-namespaces
```

## **Understanding K3s Architecture**

### **Overview**

K3s is a lightweight, production-grade Kubernetes distribution optimized for resource-constrained environments such as edge devices and IoT systems. It is designed to run as a single binary, managed by `systemd`.

### **Key Components**

- **Container Runtime**: K3s uses `containerd` as its container runtime for lightweight and efficient container management.
- **Pod Architecture**: Kubernetes pods serve as collections of containers that share network and storage resources, enabling efficient resource utilization and modular application design.

---

This document provides a technical overview and step-by-step instructions for installing and configuring K3s on a Raspberry Pi, ensuring a lightweight Kubernetes setup for edge computing environments.
