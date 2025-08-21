# A Production Kubernetes Cluster with GitOps Automation

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.30+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![K3s](https://img.shields.io/badge/K3s-Latest-FFC61E?logo=k3s&logoColor=black)](https://k3s.io/)
[![Flux](https://img.shields.io/badge/Flux-Latest-5468FF?logo=flux&logoColor=white)](https://fluxcd.io/)
[![Renovate](https://img.shields.io/badge/Renovate-Enabled-1A73E8?logo=renovatebot)](https://renovatebot.com/)

## Overview

This repository represents a **production-grade Kubernetes cluster** built on Raspberry Pi hardware, implementing enterprise-level GitOps practices for automated infrastructure and application management. The cluster demonstrates how to achieve production reliability, security, and operational excellence in a cost-effective home lab environment.

## 🏗️ Production Features

### **GitOps Automation**

- **Fully automated deployments** via Flux CD with Git as the single source of truth
- **Zero-downtime updates** through declarative configuration management
- **Automated rollbacks** and disaster recovery capabilities
- **Multi-environment support** (staging/production) with promotion workflows

### **Enterprise Security**

- **Secret encryption at rest** using SOPS with Age encryption
- **Non-root container execution** with proper security contexts
- **Network policies** and service mesh capabilities
- **TLS termination** with automated certificate management
- **Secure external access** via Cloudflare Tunnel

### **Production Monitoring & Observability**

- **Complete monitoring stack** with Prometheus, Grafana, and AlertManager
- **Real-time metrics** and alerting for infrastructure and applications
- **Log aggregation** and analysis capabilities
- **Performance monitoring** and capacity planning

### **Automated Operations**

- **Dependency management** with Renovate bot for security updates
- **Infrastructure as Code** with Kustomize and Helm
- **Automated testing** and validation pipelines
- **Self-healing** infrastructure with Kubernetes controllers

## 🚀 Architecture

This production cluster implements a robust, scalable architecture:

### **Core Infrastructure**

- **K3s Distribution:** Lightweight, production-ready Kubernetes optimized for ARM
- **GitOps Controller:** Flux CD managing continuous deployment and reconciliation
- **Service Mesh:** Integrated networking with load balancing and traffic management
- **Persistent Storage:** Distributed storage for stateful applications
- **Backup & Recovery:** Automated backup strategies for data protection

### **Application Stack**

- **Linkding:** Self-hosted bookmark manager with authentication
- **Audiobookshelf:** Media server with user management and streaming
- **Mealie:** Recipe management with meal planning capabilities
- **Monitoring Dashboard:** Grafana with custom dashboards and alerts

### **Security & Compliance**

- **Zero-trust networking** with encrypted communications
- **Role-based access control (RBAC)** for fine-grained permissions
- **Pod security standards** enforcing security policies
- **Regular security scanning** and vulnerability assessments

## 🎯 Quick Start

### Prerequisites

**Hardware Requirements:**

- Raspberry Pi 4B (4GB RAM minimum recommended)
- High-speed SD card (64GB+) or SSD storage
- Reliable network connectivity
- UPS power backup (recommended for production stability)

**Software Prerequisites:**

- Raspberry Pi OS (64-bit)
- Git, curl, and basic Linux utilities
- SSH access configured

### Deployment

1. **Bootstrap the cluster:**

```bash
# Install K3s
curl -sfL https://get.k3s.io | sh -

# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap GitOps
flux bootstrap github \
  --owner=bmacharia \
  --repository=pi-cluster \
  --branch=main \
  --path=./clusters/staging
```

2. **Verify deployment:**

```bash
# Check cluster status
kubectl get nodes
kubectl get pods -A

# Monitor Flux reconciliation
flux get kustomizations
```

3. **Access services:**

```bash
# Port forward to access applications locally
kubectl port-forward -n linkding svc/linkding 9090:9090
kubectl port-forward -n audiobookshelf svc/audiobookshelf 8080:80
kubectl port-forward -n monitoring svc/grafana 3000:80
```

## 📁 Repository Structure

```
├── 📁 apps/                    # Application deployments
│   ├── 📁 base/               # Base configurations
│   │   ├── 📁 audiobookshelf/ # Media server
│   │   ├── 📁 linkding/       # Bookmark manager
│   │   └── 📁 mealie/         # Recipe manager
│   └── 📁 staging/            # Environment overlays
├── 📁 clusters/               # Cluster configurations
│   └── 📁 staging/           # Staging environment
│       ├── apps.yaml         # Application deployments
│       ├── infrastructure.yaml # Infrastructure controllers
│       ├── monitoring.yaml   # Observability stack
│       └── 📁 flux-system/   # Flux CD configuration
├── 📁 infrastructure/         # Infrastructure controllers
│   └── 📁 controllers/       # Automated services
│       └── 📁 base/renovate/ # Dependency automation
├── 📁 monitoring/            # Observability stack
│   ├── 📁 controllers/       # Monitoring controllers
│   └── 📁 configs/          # Monitoring configurations
└── 📁 journal/              # Documentation and guides
```

## 🔐 Security & Secrets Management

This cluster implements production-grade security practices:

### **Secret Encryption**

```bash
# Secrets are encrypted using SOPS with Age
age-keygen -o age.agekey
export SOPS_AGE_KEY_FILE=age.agekey

# Example encrypted secret
sops --encrypt --age <public-key> secret.yaml > secret.enc.yaml
```

### **Security Contexts**

All containers run with:

- Non-root user execution
- Read-only root filesystems where possible
- Dropped capabilities
- Security context constraints

## 📊 Monitoring & Observability

### **Metrics & Monitoring**

- **Prometheus** for metrics collection
- **Grafana** for visualization and dashboards
- **AlertManager** for alerting and notifications
- **Node Exporter** for system metrics

### **Key Dashboards**

- Cluster overview and resource utilization
- Application performance and health
- Network traffic and security events
- Storage and backup status

## 🔄 Automated Operations

### **Dependency Management**

Renovate bot automatically:

- Scans for outdated dependencies
- Creates pull requests for updates
- Runs security vulnerability checks
- Maintains up-to-date container images

### **GitOps Workflow**

1. **Push** configuration changes to Git
2. **Flux** detects changes and reconciles
3. **Kubernetes** applies configurations
4. **Monitoring** validates deployment health

## 🛡️ Production Best Practices

### **High Availability**

- Service redundancy and load balancing
- Automated failover mechanisms
- Data replication and backup strategies
- Health checks and self-healing

### **Performance Optimization**

- Resource limits and requests configured
- HPA (Horizontal Pod Autoscaler) for scaling
- Storage optimization and caching
- Network optimization for low latency

### **Operational Excellence**

- Comprehensive logging and audit trails
- Automated testing and validation
- Regular security updates and patching
- Disaster recovery procedures

## 🚀 Scaling & Evolution

This cluster is designed for growth:

- **Multi-node expansion** capability
- **Cross-cloud deployment** patterns
- **Advanced networking** with service mesh
- **Machine learning workloads** support

## 📚 Documentation

Comprehensive guides available in `/journal/`:

- Kubernetes distribution selection
- GitOps implementation strategies
- Security hardening procedures
- Monitoring and alerting setup
- Application deployment patterns

## 🤝 Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request
4. Follow GitOps principles

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- **K3s** team for the lightweight Kubernetes distribution
- **Flux CD** community for GitOps tooling
- **Raspberry Pi Foundation** for affordable ARM hardware
- **CNCF** ecosystem for cloud-native technologies

---

**Built with ❤️ for Production Kubernetes on Edge Hardware**

🌟 _Star this repo if you find it useful!_
