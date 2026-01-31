---
name: kubespray-ha-configuration
description: Use when setting up high availability Kubernetes clusters, configuring multiple control plane nodes, etcd quorum sizing, or implementing load balancing for API server access.
---

# Kubespray HA Configuration

## Overview

High availability in Kubernetes requires redundant control plane nodes and etcd members. Kubespray automates HA setup but understanding the architecture is essential for proper sizing and troubleshooting.

**Core principle:** API servers are active-active (all handle requests). Controller-manager and scheduler use leader election (one active, others standby). etcd requires quorum (majority must agree).

## When to Use

- Deploying production clusters requiring uptime guarantees
- Sizing etcd clusters (why odd numbers matter)
- Configuring load balancing for API server
- Understanding failover behavior

**Not for:** Single-node deployments (use kubespray-deployment), troubleshooting HA failures (use kubespray-troubleshooting)

## HA Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane HA                      │
├─────────────────────────────────────────────────────────┤
│  API Server:        Active-Active (all handle requests) │
│  Controller-Mgr:    Active-Standby (leader election)    │
│  Scheduler:         Active-Standby (leader election)    │
│  etcd:              Raft consensus (quorum required)    │
└─────────────────────────────────────────────────────────┘
```

## etcd Quorum Sizing

| Nodes | Quorum | Tolerated Failures |
|-------|--------|-------------------|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Always use odd numbers.** With 4 nodes, quorum is 3 - you can still only lose 1 node (same as 3 nodes). The extra node adds complexity without fault tolerance.

## Inventory for HA Cluster

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

This is **stacked etcd** - etcd runs on control plane nodes. For **external etcd**, use separate nodes in `[etcd]` group.

## Load Balancing Options

### Option 1: Client-Side Load Balancing (Default)

Kubespray deploys nginx/haproxy on each worker node:

```
Worker Node
    └── nginx (localhost:6443)
           ├── → k8s-ctr1:6443
           ├── → k8s-ctr2:6443
           └── → k8s-ctr3:6443
```

kubelet connects to `localhost:6443`, nginx distributes to healthy API servers.

**Pros:** No external infrastructure, no single point of failure
**Cons:** Only for internal cluster traffic

Configure in `group_vars/all/all.yml`:
```yaml
loadbalancer_apiserver_type: nginx  # or haproxy
```

### Option 2: kube-vip (Software VIP)

Creates floating virtual IP that moves to healthy control plane.

**Prerequisites:**
- **ARP mode:** All nodes must be on same Layer 2 broadcast domain (typical for on-prem, NOT supported in most public clouds like AWS, GCP, Azure)
- **BGP mode:** Requires BGP-capable network infrastructure (routers that speak BGP)

```yaml
# group_vars/k8s_cluster/k8s-cluster.yml
kube_vip_enabled: true
kube_vip_arp_enabled: true      # Use ARP mode (Layer 2)
# kube_vip_bgp_enabled: true    # Use BGP mode instead
kube_vip_address: 192.168.10.100  # unused IP for VIP
```

**Pros:** Single endpoint for external access, no external LB needed
**Cons:** ARP requires Layer 2 network, BGP requires network infrastructure support

### Option 3: External Load Balancer

For cloud or enterprise environments:
- AWS: NLB/ALB
- GCP: GCP Load Balancer
- On-prem: HAProxy + keepalived

Configure endpoint in inventory:
```yaml
# group_vars/all/all.yml
loadbalancer_apiserver:
  address: 192.168.10.100
  port: 6443
```

Kubespray does NOT provision external LBs - use Terraform or manual setup.

## Verifying HA Setup

### Check Leader Election
```bash
kubectl get lease -n kube-system
# Shows kube-controller-manager and kube-scheduler leases
# holderIdentity shows current leader
```

### Check etcd Cluster Health
```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
  endpoint health
```

### Check etcd Leader
```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://192.168.10.11:2379 \
  endpoint status --write-out=table
```

## Testing Failover

### Control Plane Failover
```bash
# Stop kubelet on one control plane
ssh k8s-ctr1 "systemctl stop kubelet"

# Verify cluster still works
kubectl get nodes   # ctr1 shows NotReady
kubectl run test --image=nginx  # should succeed

# Restore
ssh k8s-ctr1 "systemctl start kubelet"
```

### etcd Failover
```bash
# Stop etcd on one node (WITH BACKUP FIRST)
ssh k8s-ctr1 "systemctl stop etcd"

# Check remaining cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-k8s-ctr2.pem \
  --key=/etc/ssl/etcd/ssl/admin-k8s-ctr2-key.pem

# Verify writes work
kubectl create configmap test-ha --from-literal=key=value

# Restore
ssh k8s-ctr1 "systemctl start etcd"
```

## Stacked vs External etcd

| Aspect | Stacked | External |
|--------|---------|----------|
| Nodes required | 3 (min HA) | 6 (3 CP + 3 etcd) |
| Complexity | Lower | Higher |
| Resource isolation | Shared | Dedicated |
| Failure domain | Coupled | Independent |

**Recommendation:** Stacked for most cases. External when etcd performance is critical or you need independent scaling.

## Common Errors (Searchable)

```
etcdserver: request timed out
```
**Cause:** etcd quorum lost (too many nodes down). **Fix:** Restore failed nodes or restore from backup.

```
kube-vip: failed to send gratuitous ARP
```
**Cause:** kube-vip ARP mode on non-Layer 2 network. **Fix:** Use BGP mode or external LB.

```
connection refused to <vip>:6443
```
**Cause:** VIP not responding. **Fix:** Check kube-vip pods, verify VIP is on correct node.

```
Unable to connect to the server: dial tcp: lookup <hostname>: no such host
```
**Cause:** DNS not resolving LB hostname. **Fix:** Use IP address or fix DNS.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Even number of etcd nodes | No additional fault tolerance |
| Missing external LB for kubectl | Can't reach API from outside cluster |
| Removing control plane without draining | Disrupted workloads |
| 2-node etcd cluster | Worse than 1 node (quorum=2, lose 1 = down) |
| kube-vip ARP in public cloud | VIP never works (no L2 broadcast) |
