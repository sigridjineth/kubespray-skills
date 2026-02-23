# Kubernetes Cluster Skills

A comprehensive skill set for deploying and managing production-ready Kubernetes clusters using [Kubespray](https://github.com/kubernetes-sigs/kubespray), [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/), and [RKE2](https://docs.rke2.io/).

## Overview

This repository contains operational skills covering the full spectrum of Kubernetes cluster lifecycle management — from bare-metal prerequisites to production monitoring and air-gapped deployments.

### Three Deployment Paths

| Path | Tool | Best For |
|------|------|----------|
| **Kubespray** | Ansible playbooks | Production HA clusters, automated deployment |
| **kubeadm** | Direct CLI | Learning, small clusters, manual control |
| **RKE2** | Rancher's distribution | Rancher ecosystem, security-focused deployments |

## Available Skills

### kubeadm (Manual Cluster Bootstrap)

| Skill | Use When |
|-------|----------|
| [kubeadm-prerequisites](kubeadm-prerequisites/SKILL.md) | Preparing nodes — containerd, kernel modules, swap, SELinux |
| [kubeadm-init](kubeadm-init/SKILL.md) | Initializing control plane — `kubeadm init`, 14 phases, certs |
| [kubeadm-join](kubeadm-join/SKILL.md) | Joining worker/control-plane nodes, TLS bootstrap, tokens |
| [kubeadm-troubleshooting](kubeadm-troubleshooting/SKILL.md) | Init failures, join failures, NotReady nodes, cert errors |

### Kubespray (Ansible-Automated Deployment)

| Skill | Use When |
|-------|----------|
| [kubespray-lab-setup](kubespray-lab-setup/SKILL.md) | Setting up Vagrant/VirtualBox lab environment |
| [kubespray-deployment](kubespray-deployment/SKILL.md) | Deploying clusters — inventory, group_vars, `cluster.yml` |
| [kubespray-troubleshooting](kubespray-troubleshooting/SKILL.md) | Deployment failures, connection refused, etcd issues |
| [kubespray-ha-configuration](kubespray-ha-configuration/SKILL.md) | HA setup, etcd sizing, load balancing strategies |
| [kubespray-operations](kubespray-operations/SKILL.md) | Upgrades, scaling, node management, etcd backup |
| [kubespray-monitoring](kubespray-monitoring/SKILL.md) | Prometheus, Grafana, etcd metrics |
| [kubespray-airgap](kubespray-airgap/SKILL.md) | Air-gapped/offline deployments |
| [kubespray-certificates](kubespray-certificates/SKILL.md) | Certificate expiration, renewal, auto-renewal |
| [kubespray-helm-airgap](kubespray-helm-airgap/SKILL.md) | Helm chart deployment in air-gapped environments |
| [kubespray-offline-infra](kubespray-offline-infra/SKILL.md) | Offline infrastructure setup (registry, file server) |

### RKE2 (Rancher Kubernetes Engine)

| Skill | Use When |
|-------|----------|
| [rke2-deployment](rke2-deployment/SKILL.md) | Deploying RKE2 clusters |
| [rke2-operations](rke2-operations/SKILL.md) | RKE2 cluster operations and maintenance |

### Other

| Skill | Use When |
|-------|----------|
| [cluster-api](cluster-api/SKILL.md) | Cluster API for declarative cluster management |

## Quick Start

### Typical kubeadm Flow

```
1. kubeadm-prerequisites  →  Prepare all nodes
2. kubeadm-init           →  Initialize control plane
3. Install CNI            →  Flannel/Calico
4. kubeadm-join           →  Join worker nodes
```

### Typical Kubespray Flow

```
1. kubespray-lab-setup         →  Provision VMs (or prepare production nodes)
2. kubespray-ha-configuration  →  Plan HA architecture
3. kubespray-deployment        →  Configure inventory and deploy
4. kubespray-certificates      →  Enable auto-renewal
5. kubespray-monitoring        →  Deploy Prometheus/Grafana
6. kubespray-operations        →  Set up backups
```

### Air-Gap Deployment Flow

```
1. kubespray-offline-infra  →  Set up registry + file server
2. kubespray-airgap         →  Stage binaries and images
3. kubespray-helm-airgap    →  Stage Helm charts
4. kubespray-deployment     →  Deploy cluster offline
```

## Common Scenarios

| Scenario | Skill |
|----------|-------|
| "Setting up a new cluster from scratch" | `kubeadm-prerequisites` → `kubeadm-init` |
| "Automated production HA deployment" | `kubespray-lab-setup` → `kubespray-deployment` |
| "kubeadm init failed" | `kubeadm-troubleshooting` |
| "Kubespray deployment failed" | `kubespray-troubleshooting` |
| "Node shows NotReady" | `kubeadm-troubleshooting` or `kubespray-troubleshooting` |
| "Token expired" | `kubeadm-join` |
| "Certificates expiring" | `kubespray-certificates` |
| "Upgrade Kubernetes version" | `kubespray-operations` |
| "Deploy to air-gapped network" | `kubespray-airgap` |
| "Set up monitoring" | `kubespray-monitoring` |
| "Deploy with RKE2" | `rke2-deployment` |

## Tested Versions

| Component | Version |
|-----------|---------|
| Kubernetes | 1.32.x |
| Kubespray | release-2.28 |
| containerd | 2.1.x |
| OS | Rocky Linux 10 / RHEL 9 |
| Ansible | 2.14+ and < 2.18 |

## Prerequisites

### For kubeadm

- SSH access to target nodes
- 2GB+ RAM, 2+ CPU cores per node
- containerd installed
- Swap disabled

### For Kubespray

- Ansible 2.14+ on control node
- Python 3.x with pip
- SSH access to all target nodes
- 2GB+ RAM, 2+ CPU cores per node

### Network Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH |
| 6443 | TCP | Kubernetes API |
| 2379-2380 | TCP | etcd |
| 10250 | TCP | kubelet |
| 10251 | TCP | kube-scheduler |
| 10252 | TCP | kube-controller-manager |
| 179 | TCP | BGP (Calico) |
| 8472 | UDP | VXLAN (Flannel) |

## Key Concepts

### The `ip=` Variable (Kubespray)

Critical for VirtualBox/multi-NIC environments — without explicit `ip=`, Kubespray may detect the wrong interface:

```ini
k8s-node1 ansible_host=192.168.10.10 ip=192.168.10.10
```

### etcd Quorum

| Nodes | Quorum | Fault Tolerance |
|-------|--------|-----------------|
| 1 | 1 | 0 (no HA) |
| 3 | 2 | 1 node |
| 5 | 3 | 2 nodes |

Always use odd numbers.

### Version Skew Policy

Kubernetes requires upgrading one minor version at a time:
```
v1.28 → v1.29 → v1.30 → v1.31  ✓
v1.28 → v1.31                   ✗
```

### Certificate Expiration

Kubespray/kubeadm certificates expire in 1 year by default. Enable auto-renewal:
```yaml
auto_renew_certificates: true  # Kubespray
```

## Directory Structure

```
kubespray-skills/
├── README.md
├── kubeadm-init/                  # kubeadm control plane init
│   └── SKILL.md
├── kubeadm-join/                  # kubeadm node joining
│   └── SKILL.md
├── kubeadm-prerequisites/         # kubeadm node preparation
│   └── SKILL.md
├── kubeadm-troubleshooting/       # kubeadm debugging
│   └── SKILL.md
├── kubespray-airgap/              # Air-gap deployment
│   └── SKILL.md
├── kubespray-certificates/        # Certificate management
│   └── SKILL.md
├── kubespray-deployment/          # Cluster deployment
│   └── SKILL.md
├── kubespray-ha-configuration/    # HA setup & load balancing
│   └── SKILL.md
├── kubespray-helm-airgap/         # Helm in air-gap
│   └── SKILL.md
├── kubespray-lab-setup/           # Vagrant lab setup
│   └── SKILL.md
├── kubespray-monitoring/          # Prometheus/Grafana
│   └── SKILL.md
├── kubespray-offline-infra/       # Offline infrastructure
│   └── SKILL.md
├── kubespray-operations/          # Upgrades & node management
│   └── SKILL.md
├── kubespray-troubleshooting/     # Kubespray debugging
│   └── SKILL.md
├── rke2-deployment/               # RKE2 deployment
│   └── SKILL.md
├── rke2-operations/               # RKE2 operations
│   └── SKILL.md
└── cluster-api/                   # Cluster API
    └── SKILL.md
```

## Installation

```bash
# Using clawhub
npx clawhub add sigridjineth/kubespray-skills

# Or manual clone
git clone https://github.com/sigridjineth/kubespray-skills.git ~/.claude/skills/kubespray-skills
```

## Related Resources

- [Kubespray GitHub](https://github.com/kubernetes-sigs/kubespray)
- [kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [RKE2 Documentation](https://docs.rke2.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Ansible Documentation](https://docs.ansible.com/)

## Contributing

1. Identify a gap or issue
2. Update the relevant `SKILL.md`
3. Test with realistic scenarios
4. Commit with descriptive message

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-01 | Initial release — 6 Kubespray skills |
| 1.1.0 | 2024-01 | Added searchable errors, fixed flowcharts |
| 2.0.0 | 2026-02 | Added kubespray-lab-setup, kubespray-monitoring. Major HA/ops/deployment updates. 8 Kubespray skills. |
| 3.0.0 | 2026-02 | Merged kubeadm skills (4 skills). Now 17 total skills covering kubeadm, Kubespray, RKE2, and Cluster API. |
