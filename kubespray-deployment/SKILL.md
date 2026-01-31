---
name: kubespray-deployment
description: Use when deploying Kubernetes clusters with Kubespray, setting up inventory files, configuring group_vars, or running cluster.yml playbook. Use when seeing "no hosts matched" or inventory parsing errors.
---

# Kubespray Deployment

## Overview

Kubespray deploys production-ready Kubernetes clusters using Ansible. It automates OS preparation, container runtime installation, etcd clustering, control plane bootstrapping, and CNI deployment.

**Core principle:** Kubespray uses kubeadm internally but handles everything kubeadm doesn't - OS config, containerd, CNI, and HA setup.

## When to Use

- Deploying new Kubernetes clusters
- Setting up inventory for multi-node clusters
- Configuring cluster variables (CNI, versions, addons)
- Running initial cluster deployment

**Not for:** Upgrades (use kubespray-operations), troubleshooting failures (use kubespray-troubleshooting)

## Quick Reference

| File | Purpose |
|------|---------|
| `inventory.ini` | Define hosts and group membership |
| `group_vars/all/*.yml` | Global settings (etcd, containerd, NTP) |
| `group_vars/k8s_cluster/*.yml` | Cluster settings (CNI, versions, addons) |
| `cluster.yml` | Main deployment playbook |

## Inventory Setup

### Critical: The `ip` Variable

```ini
# CORRECT - explicit ip prevents VirtualBox NAT trap
k8s-ctr ansible_host=192.168.10.10 ip=192.168.10.10

[kube_control_plane]
k8s-ctr

[etcd:children]
kube_control_plane

[kube_node]
k8s-ctr

[k8s_cluster:children]
kube_control_plane
kube_node
```

**Why `ip=` matters:** Without it, Kubespray may detect 10.0.2.15 (VirtualBox NAT) instead of your actual node IP. This causes `kubeadm join` to fail with "connection refused to 10.0.2.15:6443".

### Group Meanings

| Group | Contains |
|-------|----------|
| `kube_control_plane` | API server, controller-manager, scheduler nodes |
| `etcd` | etcd cluster members (odd number: 1, 3, or 5) |
| `kube_node` | Worker nodes that run pods |
| `k8s_cluster` | Union of control_plane + node (for shared config) |

## Key Variables

### group_vars/k8s_cluster/k8s-cluster.yml

```yaml
# CNI plugin (calico, flannel, cilium)
kube_network_plugin: calico

# Network CIDRs - must not overlap with existing infra
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18

# Proxy mode (ipvs recommended for scale)
kube_proxy_mode: ipvs

# Container runtime
container_manager: containerd

# Certificate auto-renewal (ENABLE THIS)
auto_renew_certificates: true
```

### group_vars/k8s_cluster/addons.yml

```yaml
# Common addons to enable
helm_enabled: true
metrics_server_enabled: true
ingress_nginx_enabled: false  # enable if needed
```

## Deployment Command

```bash
cd kubespray

# Standard deployment
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b -v

# With specific SSH key
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -u root -b -v --private-key=~/.ssh/id_rsa
```

**Flags:**
- `-b`: become (sudo)
- `-v`: verbose output
- `-i`: inventory file path (use `.ini` file, not directory)

## Execution Flow (15-30 min)

```
boilerplate.yml → facts.yml → bootstrap-os → container-engine
      ↓
   download → etcd → kubernetes/node → kubernetes/control-plane
      ↓
   kubeadm init/join → network_plugin → kubernetes-apps → done
```

## Post-Deployment Verification

```bash
# Copy kubeconfig from control plane
scp root@<control-plane-ip>:/etc/kubernetes/admin.conf ~/.kube/config

# Fix localhost reference (Linux)
sed -i 's/127.0.0.1/<control-plane-ip>/g' ~/.kube/config

# Fix localhost reference (macOS)
sed -i '' 's/127.0.0.1/<control-plane-ip>/g' ~/.kube/config

# Verify
kubectl get nodes
kubectl get pods -A
```

## Common Errors (Searchable)

```
[WARNING]: Unable to parse /path/inventory.ini as an inventory source
```
**Fix:** Check YAML/INI syntax, ensure file exists

```
[WARNING]: Could not match supplied host pattern, ignoring: etcd
skipping: no hosts matched
```
**Fix:** Use `-i inventory.ini` (file), not `-i inventory/` (directory)

```
fatal: [node]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh"}
```
**Fix:** Verify SSH key, check `ansible_host` IP, test with `ssh root@<ip>`

```
ERROR! Attempting to decrypt but no vault secrets found
```
**Fix:** Add `--ask-vault-pass` or set `ANSIBLE_VAULT_PASSWORD_FILE`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `ip=` variable | Add explicit `ip=<node-ip>` to inventory |
| Using `-i inventory/mycluster/` (directory) | Use `-i inventory/mycluster/inventory.ini` (file) |
| etcd with even nodes | Always use 1, 3, or 5 etcd nodes |
| Skipping `auto_renew_certificates` | Enable it - certificates expire in 1 year |
| Running from wrong directory | Must run from kubespray root (for ansible.cfg) |
