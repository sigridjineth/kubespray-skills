---
name: kubespray-operations
description: Use when upgrading Kubernetes versions with Kubespray, adding or removing nodes, scaling clusters, or performing etcd backup and restore operations.
---

# Kubespray Operations

## Overview

Kubespray provides playbooks for cluster lifecycle operations: upgrades, scaling, and reset. Understanding these operations prevents data loss and service disruption.

**Core principle:** Always backup etcd before destructive operations. Kubernetes upgrades must go one minor version at a time.

## When to Use

- Upgrading Kubernetes versions (patch, minor, or major with Kubespray version bump)
- Adding new worker or control plane nodes
- Removing nodes from cluster (healthy or unreachable)
- Backing up and restoring etcd
- Resetting cluster to clean state

**Not for:** Initial deployment (use kubespray-deployment), troubleshooting failures (use kubespray-troubleshooting), certificate issues (use kubespray-certificates)

## Node Management

### Adding a Worker Node (scale.yml)

**What scale.yml does:** download binaries -> install kubelet -> upload control plane certs -> kubeadm join -> apply labels/taints -> configure CNI

**IMPORTANT:** `scale.yml` is ONLY for worker nodes. Do NOT use it for control plane nodes. Control plane nodes require `cluster.yml` because they need etcd membership, static pod generation, certificate creation, and kubeadm control plane join.

**Step-by-step:**

1. Update inventory -- add the new node to `[all]` and `[kube_node]`:
```ini
[all]
# ... existing nodes ...
k8s-node5 ansible_host=192.168.10.25 ip=192.168.10.25  # new

[kube_node]
k8s-node1
k8s-node2
k8s-node3
k8s-node4
k8s-node5  # new
```

2. Run scale playbook with `--limit` targeting only the new node:
```bash
ansible-playbook scale.yml --become \
  -i inventory/mycluster/inventory.ini \
  --limit=k8s-node5 \
  -e kube_version="1.32.9"
```

3. Verify:
```bash
kubectl get nodes
# k8s-node5 should appear as Ready
```

### Removing a Worker Node (remove-node.yml)

**PDB considerations:** If any PodDisruptionBudget has `maxUnavailable: 0`, draining will block indefinitely. Always audit PDBs before removing a node:
```bash
kubectl get pdb --all-namespaces
```

**What remove-node.yml does:** confirmation prompt -> cordon and drain -> remove etcd member (if applicable) -> kubeadm reset -> delete Node object from API server

**Command:**
```bash
ansible-playbook remove-node.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e node=k8s-node5 \
  -e skip_confirmation=true
```

**After removal:** Update `inventory.ini` to remove the node entry. Kubespray does not modify your inventory file automatically.

### Force-Removing Unhealthy Nodes

When a node is unreachable (hardware crash, network partition), normal `remove-node.yml` FAILS with `UNREACHABLE` because it tries to SSH into the dead node to run `kubeadm reset`.

**Use these extra variables to force removal:**
```bash
ansible-playbook remove-node.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e node=k8s-node5 \
  -e skip_confirmation=true \
  -e reset_nodes=false \
  -e allow_ungraceful_removal=true
```

**What happens with these flags:**
- `reset_nodes=false` -- skips SSH to the dead node, skips kubeadm reset
- `allow_ungraceful_removal=true` -- skips drain (node is unreachable anyway), removes only cluster-side metadata (Node object, etcd member if applicable)

**WARNING:** If the dead node comes back online, kubelet will attempt to re-register with old certificates. You must wipe the node (kubeadm reset) before rejoining it to the cluster.

### Replacing a Control Plane Node

This is the most complex node operation because it involves etcd membership changes.

**Step 1: Remove the old control plane node**
```bash
ansible-playbook remove-node.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e node=k8s-ctr2 \
  -e skip_confirmation=true
```
This takes etcd from 3 members to 2 members. Minimize this window -- proceed to Step 2 and 3 promptly.

**Step 2: Update inventory**

**CRITICAL:** Add the new control plane node at the END of `[kube_control_plane]`. Never insert it in the middle -- Kubespray uses the ordering for etcd initial cluster membership and the first node has special significance.

```ini
[kube_control_plane]
k8s-ctr1     # existing - MUST stay first
k8s-ctr3     # existing
k8s-ctr-new  # new - added at END
```

**Step 3: Run cluster.yml (NOT scale.yml!)**

Control plane nodes need `cluster.yml` because they require:
- etcd member join
- Static pod manifest generation (kube-apiserver, kube-controller-manager, kube-scheduler)
- Full certificate generation
- kubeadm control plane join (not just worker join)

```bash
ansible-playbook cluster.yml --become \
  -i inventory/mycluster/inventory.ini
```

**CRITICAL LIMITATION:** The first node listed in `[kube_control_plane]` CANNOT be removed via `remove-node.yml`. This node is the initial etcd bootstrap node and has special handling throughout Kubespray. To replace it, you must rebuild the cluster or use manual etcd membership manipulation (advanced, not covered by standard playbooks).

**Verify replacement:**
```bash
# etcd member list should show 3 members again
ETCDCTL_API=3 etcdctl member list \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://127.0.0.1:2379

# All static pods running on new CP node
kubectl get pods -n kube-system -o wide | grep k8s-ctr-new

# nginx.conf on workers updated to include new CP endpoint
# (Kubespray uses nginx as LB on worker nodes for API server access)
```

### Node Management Key Takeaways

| Rule | Detail |
|------|--------|
| `scale.yml` for workers only | Control plane nodes require `cluster.yml` |
| New CP nodes at END of `[kube_control_plane]` | Never insert in the middle of the group |
| First CP node cannot be removed | It is the etcd bootstrap node with special handling |
| Unreachable nodes | Use `reset_nodes=false` + `allow_ungraceful_removal=true` |
| PDBs can block drains | Audit with `kubectl get pdb --all-namespaces` before removal |
| Keep inventory.ini in sync | Kubespray does not auto-update your inventory after removal |

## Cluster Upgrades

### Version Skew Policy

Kubernetes enforces strict upgrade rules -- can only upgrade one minor version at a time:
```
v1.X -> v1.X+1 -> v1.X+2  (one at a time)
v1.X -> v1.X+3             (cannot skip)
```

Check supported versions: https://kubernetes.io/releases/

### Pre-Upgrade: Flannel CNI Plugin Update

If using Flannel, update the CNI plugin BEFORE upgrading Kubernetes while all nodes are still on the same version. The Flannel DaemonSet update rolls out to all nodes at once -- you cannot do it per-node.

```bash
# Check current Flannel version
kubectl get daemonset kube-flannel -n kube-flannel \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Update the Flannel version in your Kubespray group_vars before proceeding with the Kubernetes upgrade.

### Pre-Upgrade Checklist

```bash
# 1. Check current versions
kubectl get nodes -o wide

# 2. Verify cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# 3. Audit PDBs that could block drains
kubectl get pdb --all-namespaces -o wide

# 4. Backup etcd (CRITICAL)
ETCDCTL_API=3 etcdctl snapshot save /backup/pre-upgrade.db \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://127.0.0.1:2379

# 5. Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/pre-upgrade.db
```

### Upgrade Strategies

**Strategy 1: Unsafe (cluster.yml) -- Dev/Test Only**

Uses `cluster.yml` with upgrade flag. No cordon, no drain -- workloads may be disrupted.

```bash
ansible-playbook cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  -e upgrade_cluster_setup=true
```

Only use this for development clusters where downtime is acceptable.

**Strategy 2: Graceful (upgrade-cluster.yml) -- Production**

Rolling per-node upgrade: cordon -> drain -> upgrade -> uncordon. This is the recommended approach.

```bash
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10"
```

**Serial control options:**
- `serial: 1` -- upgrades one node at a time (safest, slowest)
- `serial: "20%"` -- upgrades 20% of nodes at a time (default)

**Manual confirmation between nodes:**
```bash
# Pause and wait for operator confirmation before each node
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  -e upgrade_node_confirm=true
```

**Automatic pause between nodes:**
```bash
# Wait 60 seconds between nodes automatically
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  -e upgrade_node_pause_seconds=60
```

### Patch Upgrade (e.g., v1.32.9 -> v1.32.10)

The simplest upgrade type. Only Kubernetes binaries change. Same Kubespray version.

**Step 1: Upgrade control plane and etcd first**
```bash
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  --limit "kube_control_plane:etcd"
```

**What happens per control plane node:**
1. Pre-upgrade: downloads new binaries and images
2. Cordon the node
3. Drain workloads
4. First CP node: `kubeadm upgrade apply v1.32.10`
5. Subsequent CP nodes: `kubeadm upgrade node`
6. kube-proxy DaemonSet updated
7. Uncordon the node

**Step 2: Upgrade workers individually**
```bash
# Upgrade one worker at a time
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  --limit "k8s-node4"

# Then the next worker
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.32.10" \
  --limit "k8s-node5"
```

**Post-upgrade verification:**
```bash
# All nodes on new version
kubectl get nodes -o wide

# Static pod images updated on CP nodes
kubectl get pods -n kube-system -o wide | grep -E 'apiserver|controller|scheduler'

# kube-proxy image updated
kubectl get daemonset kube-proxy -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# etcd cluster healthy
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://127.0.0.1:2379

# API server endpoints responding
kubectl get --raw='/readyz?verbose'
```

### Minor Upgrade (e.g., v1.32.10 -> v1.33.7)

Same procedure as a patch upgrade, but with additional considerations:

- **Version skew policy:** Only one minor version jump at a time. v1.32 -> v1.33 is valid. v1.32 -> v1.34 is NOT.
- **Longer duration:** More container images to pull, more component checks
- **Update admin kubectl:** After upgrade, update the kubectl binary on your workstation to match the new cluster minor version
- **Check deprecated API usage before upgrading:**
```bash
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
```

APIs deprecated in the current version may be removed in the next minor version. Fix any usage before upgrading.

The upgrade command is the same -- just set the target version:
```bash
# CP + etcd first
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.33.7" \
  --limit "kube_control_plane:etcd"

# Then workers individually
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.33.7" \
  --limit "k8s-node4"
```

### Major Upgrade with Kubespray Version Bump (e.g., v1.33.7 -> v1.34.3)

When the target Kubernetes version requires a newer Kubespray release, you must upgrade Kubespray itself first.

**Step 1: Update Kubespray**
```bash
cd /path/to/kubespray
git fetch --all --tags
git checkout v2.30.0
```

Check the release notes for:
- Supported Kubernetes version range
- Component version changes (etcd, containerd, CNI plugins)
- Breaking changes in variable names or defaults

**Step 2: Update Python dependencies**

Use a virtual environment for dependency isolation:
```bash
python3 -m venv kubespray-venv
source kubespray-venv/bin/activate
pip3 install -r requirements.txt
```

**Step 3: Review component upgrades**

A Kubespray version bump may also upgrade:
- **etcd** (e.g., 3.5.25 -> 3.5.26): Automatic during upgrade, per-member rolling restart, backups stored in `/var/backups/`
- **containerd** (e.g., 2.1.5 -> 2.2.1): Automatic during upgrade, binary replacement + service restart

These happen transparently as part of the upgrade playbook.

**Step 4: Run the full upgrade**
```bash
# CP + etcd first
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.34.3" \
  --limit "kube_control_plane:etcd"

# Then workers
ansible-playbook upgrade-cluster.yml --become \
  -i inventory/mycluster/inventory.ini \
  -e kube_version="1.34.3" \
  --limit "k8s-node4"
```

**Step 5: Post-upgrade tasks**
```bash
# Update kubectl on your workstation to match cluster version
# Update Helm if needed

# Refresh kubeconfig if certificate contents changed
cp /etc/kubernetes/admin.conf ~/.kube/config

# Verify HAProxy/nginx LB backends are healthy (if using external LB)

# Check Prometheus targets if monitoring is deployed
# (scrape endpoints may have changed with component upgrades)

# Full verification
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get --raw='/readyz?verbose'
```

### Upgrade Order Summary

Kubespray `upgrade-cluster.yml` upgrades in this sequence:
1. etcd (if new version needed)
2. Control plane nodes (one at a time by default)
3. Worker nodes (one at a time by default)
4. CNI plugin
5. Addons

For multi-hop upgrades (e.g., 1.31 -> 1.34), run the full upgrade process three times:
```bash
# v1.31 -> v1.32
ansible-playbook upgrade-cluster.yml ... -e kube_version="1.32.x"
# verify, then v1.32 -> v1.33
ansible-playbook upgrade-cluster.yml ... -e kube_version="1.33.x"
# verify, then v1.33 -> v1.34
ansible-playbook upgrade-cluster.yml ... -e kube_version="1.34.x"
```

## etcd Backup and Restore

### Creating Backups

```bash
#!/bin/bash
# etcd-backup.sh
BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT="$BACKUP_DIR/etcd-snapshot-$DATE.db"

mkdir -p "$BACKUP_DIR"

ETCDCTL_API=3 etcdctl snapshot save "$SNAPSHOT" \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://127.0.0.1:2379

# Verify and cleanup old backups
if [ $? -eq 0 ]; then
  echo "Backup successful: $SNAPSHOT"
  find "$BACKUP_DIR" -name "*.db" -mtime +7 -delete
else
  echo "Backup failed!"
  exit 1
fi
```

### Automated Backup (systemd)

```ini
# /etc/systemd/system/etcd-backup.service
[Unit]
Description=etcd backup
After=etcd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/etcd-backup.sh

# /etc/systemd/system/etcd-backup.timer
[Unit]
Description=Daily etcd backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable etcd-backup.timer
systemctl start etcd-backup.timer
```

### Restoring from Backup (Single Node)

```bash
# 1. Stop etcd
systemctl stop etcd

# 2. Backup current data (just in case)
mv /var/lib/etcd /var/lib/etcd.broken

# 3. Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd

# 4. Fix ownership
chown -R etcd:etcd /var/lib/etcd

# 5. Start etcd
systemctl start etcd

# 6. Verify
ETCDCTL_API=3 etcdctl endpoint health ...
kubectl get nodes
```

### Restoring Multi-Node Cluster

More complex -- each node needs restore with cluster membership info:

```bash
# On each etcd node:
ETCDCTL_API=3 etcdctl snapshot restore /backup/snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=k8s-ctr1 \
  --initial-cluster=k8s-ctr1=https://192.168.10.11:2380,k8s-ctr2=https://192.168.10.12:2380,k8s-ctr3=https://192.168.10.13:2380 \
  --initial-cluster-token=etcd-cluster-restore \
  --initial-advertise-peer-urls=https://192.168.10.11:2380
```

Use different `--initial-cluster-token` than original to prevent confusion.

## Cluster Reset

**Warning:** Destroys all cluster data including etcd.

```bash
ansible-playbook -i inventory/mycluster/inventory.ini reset.yml -b
# Type "yes" when prompted
```

Use reset when:
- Deployment is corrupted beyond repair
- Starting fresh with new configuration
- Cleaning up test clusters

## Playbook Reference

| Playbook | Purpose | When to Use |
|----------|---------|-------------|
| `cluster.yml` | Initial deployment or add CP nodes | New cluster, or adding control plane nodes |
| `upgrade-cluster.yml` | Graceful version upgrades | Production upgrades (cordon/drain/upgrade/uncordon) |
| `scale.yml` | Add worker nodes | Adding workers only (NOT for control plane) |
| `remove-node.yml` | Remove nodes | Removing any node from the cluster |
| `reset.yml` | Destroy cluster | Full cluster teardown |
| `recover-control-plane.yml` | Restore failed control plane | Control plane recovery after failure |

## Common Errors (Searchable)

```
error: unable to upgrade connection: pod does not exist
```
**Cause:** Node drained but pods not fully terminated. **Fix:** Wait and retry drain.

```
The connection to the server was refused - did you specify the right host or port?
```
**Cause:** API server down during upgrade. **Fix:** Wait for control plane to recover.

```
etcdserver: mvcc: database space exceeded
```
**Cause:** etcd database full. **Fix:** Compact and defrag etcd before upgrade.

```
UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```
**Cause:** Previous upgrade incomplete. **Fix:** Check Helm releases, clean up stuck releases.

```
cannot exec into a container in a completed pod
```
**Cause:** Pods terminated during drain. **Fix:** Normal during drain, continue.

```
UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh"}
```
**Cause:** Node is down or network partitioned during remove-node.yml. **Fix:** Use `-e reset_nodes=false -e allow_ungraceful_removal=true` to force removal from cluster metadata only.

```
cannot evict pod as it would violate the pod's disruption budget
```
**Cause:** PDB with `maxUnavailable: 0` blocks drain indefinitely. **Fix:** Audit PDBs with `kubectl get pdb --all-namespaces`, temporarily adjust or delete the blocking PDB, then retry.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Skipping Kubernetes versions | Upgrade fails, potential cluster corruption |
| No etcd backup before upgrade | Cannot recover if upgrade fails |
| Removing etcd nodes without quorum check | Cluster becomes unavailable |
| Using reset.yml when scale would work | Unnecessary downtime and data loss |
| Draining without `--ignore-daemonsets` | Drain hangs waiting for DS pods |
| Using scale.yml for control plane nodes | CP node joins as worker, missing etcd and static pods |
| Inserting new CP node in middle of inventory | Breaks etcd membership ordering assumptions |
| Trying to remove first CP node via remove-node.yml | Operation not supported, cluster may break |
| Not updating inventory.ini after remove-node.yml | Next playbook run has stale node references |
| Ignoring PDB audit before node removal | Drain hangs indefinitely on maxUnavailable: 0 |
| Dead node rejoining without wipe after force removal | Old certs conflict, kubelet in bad state |
