---
name: kubespray-operations
description: Use when upgrading Kubernetes versions with Kubespray, adding or removing nodes, scaling clusters, or performing etcd backup and restore operations.
---

# Kubespray Operations

## Overview

Kubespray provides playbooks for cluster lifecycle operations: upgrades, scaling, and reset. Understanding these operations prevents data loss and service disruption.

**Core principle:** Always backup etcd before destructive operations. Kubernetes upgrades must go one minor version at a time.

## When to Use

- Upgrading Kubernetes versions
- Adding new worker or control plane nodes
- Removing nodes from cluster
- Backing up and restoring etcd
- Resetting cluster to clean state

**Not for:** Initial deployment (use kubespray-deployment), troubleshooting failures (use kubespray-troubleshooting), certificate issues (use kubespray-certificates)

## Cluster Upgrades

### Version Skew Policy

Kubernetes enforces strict upgrade rules - can only upgrade one minor version at a time:
```
v1.X → v1.X+1 → v1.X+2  ✓ (one at a time)
v1.X → v1.X+3           ✗ (cannot skip)
```

Check supported versions: https://kubernetes.io/releases/

### Pre-Upgrade Checklist

```bash
# 1. Check current versions
kubectl get nodes -o wide

# 2. Verify cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# 3. Backup etcd (CRITICAL)
ETCDCTL_API=3 etcdctl snapshot save /backup/pre-upgrade.db \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://127.0.0.1:2379

# 4. Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/pre-upgrade.db
```

### Upgrade Command

```bash
# Upgrade to specific version
ansible-playbook -i inventory/mycluster/inventory.ini upgrade-cluster.yml \
  -e kube_version=v1.31.0 -b

# For multi-hop upgrade (1.30 → 1.33), run three times:
ansible-playbook ... -e kube_version=v1.31.0
# verify, then:
ansible-playbook ... -e kube_version=v1.32.0
# verify, then:
ansible-playbook ... -e kube_version=v1.33.0
```

### Upgrade Order

Kubespray upgrades in this sequence:
1. etcd (if new version needed)
2. Control plane nodes (one at a time)
3. Worker nodes (one at a time)
4. CNI plugin
5. Addons

### Post-Upgrade Verification

```bash
kubectl get nodes -o wide  # Check versions
kubectl get pods -A        # All pods running

# Modern health check (Kubernetes 1.19+)
kubectl get --raw='/readyz?verbose'

# Legacy component status (deprecated, may not work)
kubectl get cs
```

## Scaling Nodes

### Adding Worker Nodes

1. Update inventory with new node:
```ini
[all]
# ... existing nodes ...
k8s-w3 ansible_host=192.168.10.23 ip=192.168.10.23  # new

[kube_node]
k8s-w1
k8s-w2
k8s-w3  # new
```

2. Run scale playbook with limit:
```bash
ansible-playbook -i inventory/mycluster/inventory.ini scale.yml \
  --limit=k8s-w3 -b
```

3. Verify:
```bash
kubectl get nodes
```

### Removing Worker Nodes

1. Drain the node:
```bash
kubectl drain k8s-w1 --ignore-daemonsets --delete-emptydir-data
```

2. Remove from cluster:
```bash
ansible-playbook -i inventory/mycluster/inventory.ini remove-node.yml \
  -e node=k8s-w1 -b
```

3. Update inventory (remove the node entry)

### Adding Control Plane Nodes

Same as worker nodes, but add to `[kube_control_plane]` group. Scale playbook handles kubeadm join with `--control-plane` flag.

**Warning:** Adding control plane nodes to running cluster is complex. Test in non-production first.

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

More complex - each node needs restore with cluster membership info:

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

| Playbook | Purpose |
|----------|---------|
| `cluster.yml` | Initial deployment |
| `upgrade-cluster.yml` | Version upgrades |
| `scale.yml` | Add nodes |
| `remove-node.yml` | Remove nodes |
| `reset.yml` | Destroy cluster |
| `recover-control-plane.yml` | Restore failed control plane |

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

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Skipping Kubernetes versions | Upgrade fails, potential cluster corruption |
| No etcd backup before upgrade | Cannot recover if upgrade fails |
| Removing etcd nodes without quorum check | Cluster becomes unavailable |
| Using reset.yml when scale would work | Unnecessary downtime and data loss |
| Draining without `--ignore-daemonsets` | Drain hangs waiting for DS pods |
