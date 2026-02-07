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

## Ansible Variable Precedence

Understanding precedence is essential -- setting a variable in the wrong place means it silently has no effect.

```
1. Role defaults         (roles/xxx/defaults/main.yml)          <- Lowest priority
2. Role vars             (roles/xxx/vars/main.yml)
3. Inventory group_vars  (inventory/mycluster/group_vars/...)
4. Inventory host_vars   (inventory/mycluster/host_vars/...)
5. Playbook vars         (vars: section in playbook YAML)
6. CLI extra-vars        (--extra-vars / -e on command line)    <- Highest priority
```

**How this works in practice:**
- **Role defaults** (`defaults/main.yml`) are the baseline values Kubespray ships. These are designed to be overridden.
- **`group_vars/`** is where most customization happens. This is the standard place to set CNI, versions, addons, and network CIDRs.
- **`-e` (extra-vars) always wins.** It overrides everything, including role vars. Use it for one-off overrides or CI/CD pipelines.

**Common mistake:** A variable set in `group_vars/` has no effect because the same variable is defined in `roles/xxx/vars/main.yml` (precedence level 2), which is higher than `group_vars` (level 3). The fix is to use `-e` to force the override.

```bash
# Override any variable regardless of where it is defined
ansible-playbook cluster.yml -e kube_version="v1.32.9"
```

## Inventory Setup

### Critical: The `ip` Variable and Dual-Network Design

Kubespray uses a dual-network design for each node:
- **`ansible_host`** -- the management IP that Ansible uses to SSH into the node
- **`ip`** -- the IP address used for Kubernetes internal communication (etcd, API server advertise address, kubelet, CNI)

In many environments these are the same IP, but in multi-NIC setups (VirtualBox, cloud instances with public/private IPs) they can differ.

```ini
# Dual-network: ansible_host for SSH, ip for K8s internal traffic
k8s-node1 ansible_host=192.168.56.11 ip=10.0.1.11
k8s-node2 ansible_host=192.168.56.12 ip=10.0.1.12
k8s-node3 ansible_host=192.168.56.13 ip=10.0.1.13

# Single-network: same IP for both (still set ip= explicitly)
k8s-node1 ansible_host=192.168.10.10 ip=192.168.10.10
k8s-node2 ansible_host=192.168.10.11 ip=192.168.10.11
k8s-node3 ansible_host=192.168.10.12 ip=192.168.10.12

[kube_control_plane]
k8s-node1
k8s-node2
k8s-node3

[etcd:children]
kube_control_plane

[kube_node]
k8s-node1
k8s-node2
k8s-node3

[k8s_cluster:children]
kube_control_plane
kube_node
```

**Why `ip=` matters:** Without it, Kubespray may detect 10.0.2.15 (VirtualBox NAT) instead of your actual node IP. This causes `kubeadm join` to fail with "connection refused to 10.0.2.15:6443".

### Group Meanings and Inheritance

| Group | Contains |
|-------|----------|
| `kube_control_plane` | API server, controller-manager, scheduler nodes |
| `etcd` | etcd cluster members (odd number: 1, 3, or 5) |
| `kube_node` | Worker nodes that run pods |
| `k8s_cluster` | Union of control_plane + node (for shared config) |

**`[etcd:children]` with `kube_control_plane` -- "stacked etcd" pattern:**
Using `[etcd:children]` followed by `kube_control_plane` means every control plane node also runs etcd. This is the "stacked etcd" topology where etcd is co-located with the API server. The alternative is external etcd (separate `[etcd]` group with dedicated hosts), but stacked is the most common pattern for simplicity.

**Node ordering matters:** The first node listed in `[kube_control_plane]` is the initial control plane node during `kubeadm init`. All other control plane nodes join it with `kubeadm join --control-plane`. If that first node is unreachable during initial deployment, the entire cluster bootstrap fails.

**`[k8s_cluster:children]` as convenience aggregate:** This group exists so that `group_vars/k8s_cluster/` applies to both control plane and worker nodes. Kubernetes-wide settings (CNI, kube_version, service CIDRs) go here instead of duplicating them in separate groups.

## Key Variables

### group_vars/k8s_cluster/k8s-cluster.yml

```yaml
# Kubernetes version
kube_version: v1.32.9

# CNI plugin (calico, flannel, cilium)
kube_network_plugin: flannel

# Network CIDRs - must not overlap with existing infra
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18

# Proxy mode
# iptables: simpler, sufficient for small clusters
# ipvs: O(1) rule lookup, better for large scale (100+ services)
kube_proxy_mode: iptables

# Container runtime
container_manager: containerd

# Certificate auto-renewal (ENABLE THIS)
auto_renew_certificates: true

# DNS caching on each node (optional)
enable_nodelocaldns: false
```

### CNI-Specific: Flannel

```yaml
# CRITICAL for multi-NIC environments (e.g., VirtualBox with NAT + host-only)
# Without this, flannel may pick the NAT interface (10.0.2.15) and pod networking breaks
flannel_interface: enp0s9
```

If nodes have multiple NICs, flannel will auto-detect the interface -- and it often picks the wrong one (the NAT/default-route interface). Setting `flannel_interface` explicitly ensures flannel uses the correct network for pod-to-pod communication.

### Kubespray DNS Defaults

Kubespray sets clusterDNS to **10.233.0.3** (NOT the kubeadm default of 10.96.0.10 or 10.233.0.10). This is the third IP in the `kube_service_addresses` range (10.233.0.0/18). CoreDNS pods serve at this address.

### group_vars/k8s_cluster/addons.yml

```yaml
# Common addons to enable
helm_enabled: true

# Required for kubectl top and HPA (Horizontal Pod Autoscaler)
metrics_server_enabled: true

ingress_nginx_enabled: false  # enable if needed
```

### group_vars/all/etcd.yml

```yaml
# Enable etcd metrics endpoint (useful for later Prometheus monitoring)
etcd_metrics: basic
etcd_listen_metrics_urls: "http://0.0.0.0:2381"
```

## Deployment Command

```bash
cd kubespray

# Standard deployment
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b -v

# With specific SSH key
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -u root -b -v --private-key=~/.ssh/id_rsa

# Override a variable at deploy time
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -b -v -e kube_version="v1.32.9"
```

**Flags:**
- `-b`: become (sudo)
- `-v`: verbose output
- `-i`: inventory file path (use `.ini` file, not directory)
- `-e`: extra-vars override (highest precedence)

## Deployment Phases (~8 min in lab environment)

```
Phase 1: Prerequisites
   OS validation, kernel modules (br_netfilter, overlay), sysctl
   (net.bridge.bridge-nf-call-iptables, ip_forward), swap disabled
         |
Phase 2: Container Runtime
   containerd install + config (/etc/containerd/config.toml)
         |
Phase 3: Downloads
   kubeadm/kubelet/kubectl binaries -> /tmp/releases/
   Container images pulled (pause, etcd, API server, etc.)
         |
Phase 4: etcd Cluster Bootstrap
   etcd init on first node -> join remaining etcd members
   Verify cluster health (3 members, 1 leader, same RAFT term)
         |
Phase 5: Control Plane Init
   kubeadm init on first [kube_control_plane] node
   kubeadm join --control-plane on remaining CP nodes
         |
Phase 6: Worker Join
   kubelet + certs provisioned on workers
   kubeadm join (worker mode)
   nginx-proxy deployed on workers (for HA API access)
         |
Phase 7: CNI + Addons
   Flannel/Calico/Cilium DaemonSet deployed
   CoreDNS, kube-proxy, Metrics Server, any enabled addons
         |
       Done
```

## Post-Deployment Verification

### 1. Copy kubeconfig

```bash
# Copy kubeconfig from control plane
scp root@<control-plane-ip>:/etc/kubernetes/admin.conf ~/.kube/config

# Fix localhost reference (Linux)
sed -i 's/127.0.0.1/<control-plane-ip>/g' ~/.kube/config

# Fix localhost reference (macOS)
sed -i '' 's/127.0.0.1/<control-plane-ip>/g' ~/.kube/config

# Verify basic access
kubectl get nodes
```

### 2. System Pod Health Check

```bash
kubectl get pods -n kube-system -o wide
```

What to look for in healthy output:

| Pod | Expected Count | Runs On |
|-----|---------------|---------|
| kube-apiserver | 1 per control plane node | CP nodes (static pod) |
| kube-controller-manager | 1 per control plane node | CP nodes (static pod) |
| kube-scheduler | 1 per control plane node | CP nodes (static pod) |
| etcd | 1 per etcd member | etcd nodes (static pod) |
| flannel (or calico/cilium) | 1 per node (DaemonSet) | All nodes |
| kube-proxy | 1 per node (DaemonSet) | All nodes |
| coredns | 2 replicas (default) | Any schedulable nodes |
| metrics-server | 1 replica | Any schedulable node |
| nginx-proxy | 1 per worker node | Worker nodes only |
| nodelocaldns | 1 per node (if enabled) | All nodes |

All pods should show `Running` status with `READY` columns showing full readiness (e.g., `1/1`).

### 3. etcd Cluster Health

```bash
# SSH to a control plane node, then:
etcdctl.sh member list -w table
etcdctl.sh endpoint status -w table
```

Verify:
- All 3 (or 1, or 5) members show `started` status
- Exactly one member is `leader: true`
- All members share the same **RAFT TERM** (indicates no split-brain or re-elections)

### 4. API Server Endpoint Verification

```bash
# Test each control plane node's API server directly
curl -sk https://192.168.10.10:6443/version
curl -sk https://192.168.10.11:6443/version
curl -sk https://192.168.10.12:6443/version
```

Each should return a JSON response with `gitVersion` matching the deployed `kube_version`.

### 5. Container Images and Binaries

```bash
# Check pulled images on a node
ssh k8s-node1 crictl images

# Check downloaded binaries
ls -la /tmp/releases/
# Should contain: kubeadm, kubectl, kubelet
```

### 6. Certificate Inspection

```bash
# On a control plane node
kubeadm certs check-expiration
```

Verify certificates are freshly issued (expiry ~1 year from now). With `auto_renew_certificates: true`, Kubespray will handle renewal automatically.

### 7. CoreDNS Verification

```bash
# Test DNS resolution inside the cluster
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```

Expected output:
- Server: **10.233.0.3** (Kubespray's clusterDNS)
- Address: **10.233.0.1** (the kubernetes service ClusterIP, first IP in service CIDR)

If DNS resolution fails, check CoreDNS pod logs and ensure the CNI is healthy (pods need networking to reach CoreDNS).

### 8. Metrics Server

```bash
# Should return CPU/memory for all nodes (may take 1-2 min after deploy)
kubectl top nodes

# Should return CPU/memory for pods
kubectl top pods -A
```

If `kubectl top` returns "metrics not available", wait 60 seconds for metrics-server to collect its first scrape.

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
| Variable in group_vars has no effect | Check if same var exists in role vars (higher precedence); use `-e` to override |
| Flannel picks wrong NIC in multi-NIC setup | Set `flannel_interface: <correct-interface>` explicitly |
| DNS test resolves to wrong IP | Kubespray uses 10.233.0.3, not kubeadm default 10.96.0.10 |
