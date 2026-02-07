# Kubespray Skills

A comprehensive skill set for deploying and managing production-ready Kubernetes clusters using [Kubespray](https://github.com/kubernetes-sigs/kubespray).

## Overview

Kubespray is an Ansible-based tool that automates Kubernetes cluster deployment. It handles everything that `kubeadm` doesn't: OS preparation, container runtime installation, CNI networking, etcd clustering, and high availability setup.

These skills encode operational knowledge for:
- Lab environment setup (Vagrant/VirtualBox)
- First-time cluster deployments
- Troubleshooting common failures
- High availability configuration
- Cluster lifecycle operations (upgrades, scaling, node management)
- Production monitoring (Prometheus, Grafana, etcd metrics)
- Air-gapped/offline deployments
- Certificate management

## How to Use

### Automatic Invocation

Skills are automatically loaded by Claude Code when your task matches the skill's triggers. Simply describe what you need:

| Your Request | Skill Invoked |
|--------------|---------------|
| "Set up a Vagrant lab for Kubespray" | `kubespray-lab-setup` |
| "Deploy a Kubernetes cluster with Kubespray" | `kubespray-deployment` |
| "My deployment failed with connection refused" | `kubespray-troubleshooting` |
| "Set up HA with multiple control planes" | `kubespray-ha-configuration` |
| "Upgrade from Kubernetes 1.32 to 1.34" | `kubespray-operations` |
| "Set up Prometheus and Grafana monitoring" | `kubespray-monitoring` |
| "Deploy to air-gapped environment" | `kubespray-airgap` |
| "Certificates are expiring" | `kubespray-certificates` |

### Direct Invocation

Invoke a skill directly by name:
```
/kubespray-lab-setup
/kubespray-deployment
/kubespray-troubleshooting
/kubespray-ha-configuration
/kubespray-operations
/kubespray-monitoring
/kubespray-airgap
/kubespray-certificates
```

## Available Skills

### kubespray-lab-setup

**Use when:** Setting up a local Vagrant/VirtualBox lab environment for Kubespray, provisioning multi-node clusters with Rocky Linux 10.

**Covers:**
- Architecture overview (admin-lb + 3 CP + 2 workers)
- Complete Vagrantfile with networking configuration
- Admin-LB bootstrap script (HAProxy, NFS, tools, SSH keys)
- Node init script (swap, kernel modules, sysctl)
- Rocky Linux 10 considerations (Python 3.12 PEP 668)
- Environment deployment and verification

---

### kubespray-deployment

**Use when:** Deploying new Kubernetes clusters, setting up inventory files, configuring group_vars, or running `cluster.yml`.

**Covers:**
- Inventory file structure and syntax
- The critical `ip=` variable (VirtualBox NAT trap)
- Group meanings (`kube_control_plane`, `etcd`, `kube_node`)
- Key variables in `k8s-cluster.yml` and `addons.yml`
- Deployment commands and flags
- Post-deployment verification

---

### kubespray-troubleshooting

**Use when:** Deployment fails, `kubeadm join` errors, etcd health check failures, nodes stuck NotReady, or certificate errors.

**Covers:**
- Quick diagnostic flow (which task failed → what to check)
- VirtualBox NAT IP trap (10.0.2.15 connection refused)
- etcd health check failures
- Nodes stuck in NotReady state
- "No hosts matched" inventory errors
- Container runtime not running
- Reset and recovery procedures
- Log locations for all components

---

### kubespray-ha-configuration

**Use when:** Setting up high availability clusters, sizing etcd, or configuring load balancing.

**Covers:**
- HA architecture (active-active API servers, leader election)
- etcd quorum sizing (why odd numbers matter)
- Stacked vs external etcd
- Three detailed load balancing cases:
  - Case 1: Client-side NGINX proxy (default)
  - Case 2: External HAProxy + client-side NGINX (recommended)
  - Case 3: External LB as single endpoint
- Certificate SAN updates for external LB
- Failure simulation procedures for each case
- Choosing the right configuration

---

### kubespray-operations

**Use when:** Upgrading Kubernetes versions, adding/removing nodes, or performing etcd backup/restore.

**Covers:**
- Version skew policy (one minor version at a time)
- Node management: adding workers (scale.yml), removing nodes, force-removing unhealthy nodes, replacing control plane nodes
- Upgrade strategies: unsafe (cluster.yml) vs graceful (upgrade-cluster.yml)
- Patch, minor, and major upgrade walkthroughs (v1.32 → v1.34)
- Kubespray version bump procedures (git checkout, Python deps)
- etcd and containerd version upgrades
- etcd backup scripts and automation
- PodDisruptionBudget considerations
- Cluster reset

---

### kubespray-monitoring

**Use when:** Setting up Prometheus, Grafana, and Alertmanager on Kubespray clusters, configuring etcd metrics, or deploying NFS storage provisioners.

**Covers:**
- NFS subdir external provisioner for persistent storage
- kube-prometheus-stack Helm chart with custom values
- Grafana dashboard import via ConfigMap sidecar
- Enabling etcd metrics (etcd_listen_metrics_urls)
- Sample PromQL queries for etcd health
- HAProxy and etcd scrape configuration
- Complete monitoring target verification

---

### kubespray-airgap

**Use when:** Deploying to networks with no internet access.

**Covers:**
- Infrastructure requirements (registry, file server, control node)
- Identifying required binaries and images
- Generating complete image lists
- Staging binaries on internal HTTP server
- Populating private container registry
- Configuring containerd registry mirrors
- Running offline deployments
- Package manager dependencies

---

### kubespray-certificates

**Use when:** Certificates are expiring, checking expiration dates, or setting up auto-renewal.

**Covers:**
- Certificate types and locations
- Checking expiration with kubeadm and openssl
- Enabling auto-renewal (systemd timer)
- Manual renewal procedures
- etcd certificate renewal
- Troubleshooting x509 errors
- Monitoring certificate expiration

## Prerequisites

### Control Node (where you run Ansible)

| Requirement | Version |
|-------------|---------|
| Ansible | 2.14+ and < 2.18 |
| Python | 3.x |
| pip | Latest |

### Target Nodes

| Requirement | Details |
|-------------|---------|
| OS | Rocky Linux, Ubuntu, Debian, CentOS, etc. |
| SSH | Passwordless SSH access from control node |
| Python | Python 3 installed |
| Memory | 2GB+ RAM per node (4GB+ recommended) |
| CPU | 2+ cores per node |
| Disk | 20GB+ free space |

### Network

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

## Quick Start

### 1. Clone Kubespray

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout release-2.28  # Use latest stable release
```

### 2. Install Dependencies

```bash
# Using pip (recommended)
pip install -r requirements.txt

# Or using virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 3. Create Inventory

```bash
# Copy sample inventory
cp -rfp inventory/sample inventory/mycluster

# Edit inventory file
vim inventory/mycluster/inventory.ini
```

**Example single-node inventory:**
```ini
[all]
k8s-node1 ansible_host=192.168.10.10 ip=192.168.10.10

[kube_control_plane]
k8s-node1

[etcd]
k8s-node1

[kube_node]
k8s-node1

[k8s_cluster:children]
kube_control_plane
kube_node
```

**Example HA cluster inventory:**
```ini
[all]
k8s-ctr1 ansible_host=192.168.10.11 ip=192.168.10.11
k8s-ctr2 ansible_host=192.168.10.12 ip=192.168.10.12
k8s-ctr3 ansible_host=192.168.10.13 ip=192.168.10.13
k8s-w1 ansible_host=192.168.10.21 ip=192.168.10.21
k8s-w2 ansible_host=192.168.10.22 ip=192.168.10.22

[kube_control_plane]
k8s-ctr1
k8s-ctr2
k8s-ctr3

[etcd]
k8s-ctr1
k8s-ctr2
k8s-ctr3

[kube_node]
k8s-w1
k8s-w2

[k8s_cluster:children]
kube_control_plane
kube_node
```

### 4. Configure Variables (Optional)

```bash
# Edit cluster configuration
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

**Key variables:**
```yaml
# CNI plugin
kube_network_plugin: calico  # or flannel, cilium

# Enable certificate auto-renewal (RECOMMENDED)
auto_renew_certificates: true

# Proxy mode
kube_proxy_mode: ipvs
```

### 5. Deploy

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b
```

### 6. Access Cluster

```bash
# Copy kubeconfig
scp root@192.168.10.10:/etc/kubernetes/admin.conf ~/.kube/config

# Update server address if needed
sed -i 's/127.0.0.1/192.168.10.10/g' ~/.kube/config

# Verify
kubectl get nodes
kubectl get pods -A
```

## Common Workflows

### Workflow: Lab Environment Setup

```
1. kubespray-lab-setup     → Provision Vagrant/VirtualBox VMs
2. kubespray-deployment    → Set up inventory and variables
3. Run cluster.yml         → Deploy cluster
4. kubespray-troubleshooting → If deployment fails
5. Verify with kubectl     → Confirm success
```

### Workflow: Production HA Setup

```
1. kubespray-lab-setup           → Provision lab (or prepare production nodes)
2. kubespray-ha-configuration    → Plan HA architecture and LB strategy
3. kubespray-deployment          → Configure inventory for HA
4. Run cluster.yml               → Deploy
5. kubespray-certificates        → Enable auto-renewal
6. kubespray-monitoring          → Deploy Prometheus/Grafana stack
7. kubespray-operations          → Set up etcd backups
```

### Workflow: Cluster Upgrade

```
1. kubespray-operations    → Pre-upgrade checklist
2. kubespray-operations    → Backup etcd
3. Run upgrade-cluster.yml → Upgrade (one version at a time)
4. Verify                  → Check nodes and pods
5. Repeat for each version
```

### Workflow: Air-Gap Deployment

```
1. kubespray-airgap      → Identify required files
2. Stage binaries        → Internal HTTP server
3. Stage images          → Private registry
4. kubespray-airgap      → Configure offline.yml
5. kubespray-deployment  → Deploy cluster
```

### Workflow: Production Monitoring

```
1. kubespray-monitoring    → Deploy NFS provisioner for storage
2. kubespray-monitoring    → Install kube-prometheus-stack
3. kubespray-monitoring    → Enable etcd metrics collection
4. kubespray-monitoring    → Import Grafana dashboards
5. Verify Prometheus targets → All targets UP
```

### Workflow: Troubleshooting

```
1. kubespray-troubleshooting → Identify failure
2. Check logs                → journalctl, kubectl logs
3. Fix issue                 → Based on diagnosis
4. Re-run playbook          → Ansible is idempotent
5. kubespray-troubleshooting → If still failing, try reset
```

## Key Concepts

### The `ip=` Variable

**Critical for VirtualBox/multi-NIC environments.** Without explicit `ip=`, Kubespray may detect the wrong interface (e.g., 10.0.2.15 NAT adapter), causing `kubeadm join` to fail.

```ini
# ALWAYS specify ip= explicitly
k8s-node1 ansible_host=192.168.10.10 ip=192.168.10.10
```

### etcd Quorum

etcd uses Raft consensus requiring a majority (quorum) to operate:

| Nodes | Quorum | Fault Tolerance |
|-------|--------|-----------------|
| 1 | 1 | 0 (no HA) |
| 3 | 2 | 1 node |
| 5 | 3 | 2 nodes |
| 7 | 4 | 3 nodes |

**Always use odd numbers.** Even numbers provide no additional fault tolerance.

### Version Skew Policy

Kubernetes requires upgrading one minor version at a time:
```
v1.28 → v1.29 → v1.30 → v1.31  ✓
v1.28 → v1.31                   ✗
```

### Certificate Expiration

Kubespray certificates expire in **1 year** by default. Enable auto-renewal:
```yaml
auto_renew_certificates: true
```

Or face cluster outage when certificates expire.

## Troubleshooting Quick Reference

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| `connection refused to 10.0.2.15:6443` | VirtualBox NAT IP | Add `ip=` to inventory |
| `no hosts matched` | Wrong inventory path | Use `-i file.ini` not `-i directory/` |
| `etcd health check failed` | Wrong IP or stale state | Reset and redeploy |
| `nodes NotReady` | CNI not installed | Check network_plugin role |
| `x509 certificate expired` | Certs expired | Run `kubeadm certs renew all` |
| `container runtime not running` | containerd stopped | `systemctl restart containerd` |

## Directory Structure

```
kubespray-skills/
├── README.md                           # This file
├── kubespray-lab-setup/
│   └── SKILL.md                        # Vagrant/VirtualBox lab setup
├── kubespray-deployment/
│   └── SKILL.md                        # Deployment guide
├── kubespray-troubleshooting/
│   └── SKILL.md                        # Troubleshooting guide
├── kubespray-ha-configuration/
│   └── SKILL.md                        # HA setup & load balancing guide
├── kubespray-operations/
│   └── SKILL.md                        # Node management & upgrades guide
├── kubespray-monitoring/
│   └── SKILL.md                        # Prometheus/Grafana monitoring guide
├── kubespray-airgap/
│   └── SKILL.md                        # Air-gap deployment guide
└── kubespray-certificates/
    └── SKILL.md                        # Certificate management guide
```

## Related Resources

- [Kubespray GitHub](https://github.com/kubernetes-sigs/kubespray)
- [Kubespray Documentation](https://kubespray.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Ansible Documentation](https://docs.ansible.com/)
- [etcd Documentation](https://etcd.io/docs/)

## Contributing

To improve these skills:

1. Identify a gap or issue
2. Update the relevant `SKILL.md` file
3. Follow the skill writing guidelines:
   - Use "Use when..." in descriptions
   - Include searchable error messages
   - Add "Not for:" guidance
   - Include code examples
   - Add common mistakes table
4. Test the skill with real scenarios
5. Commit with descriptive message

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-01 | Initial release with 6 skills |
| 1.1.0 | 2024-01 | Added searchable errors, fixed flowcharts, enhanced documentation |
| 2.0.0 | 2026-02 | Added kubespray-lab-setup and kubespray-monitoring skills. Major updates to kubespray-ha-configuration (3 detailed LB cases with HAProxy), kubespray-operations (node management, patch/minor/major upgrade walkthroughs), and kubespray-deployment (variable precedence, deployment phases, expanded validation). Now 8 skills total. |
