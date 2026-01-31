---
name: kubespray-troubleshooting
description: Use when Kubespray deployment fails, kubeadm join errors, etcd health check failures, connection refused to 10.0.2.15, nodes stuck NotReady, or certificate errors during deployment.
---

# Kubespray Troubleshooting

## Overview

Diagnose and fix common Kubespray deployment failures. Most failures stem from network misconfiguration, etcd issues, or stale state from previous attempts.

**Core principle:** Read the exact task name that failed, check logs on that specific node, then fix and re-run (Ansible is idempotent).

## When to Use

- Deployment fails mid-playbook
- `kubeadm join` errors
- etcd health check timeouts
- Nodes stuck in NotReady state
- Certificate-related failures

**Not for:** Initial deployment setup (use kubespray-deployment), upgrades (use kubespray-operations), certificate renewal (use kubespray-certificates)

## Quick Diagnostic Flow

```
Playbook failed
       │
       ▼
┌─────────────────┐
│  Which task?    │
└────────┬────────┘
         │
    ┌────┼────┬────────────┐
    │    │    │            │
    ▼    ▼    ▼            ▼
  etcd  join  containerd  other
    │    │    │            │
    ▼    ▼    ▼            ▼
  Check  Check  Check    Check
  etcd   IP     containerd Ansible
  logs   config status    logs -vvv
```

| Task Failed | First Check | Command |
|-------------|-------------|---------|
| etcd health | etcd logs | `journalctl -u etcd -f` |
| kubeadm join | IP configuration | Verify `ip=` in inventory |
| container-engine | containerd status | `systemctl status containerd` |
| download | Network/proxy | Check internet connectivity |
| any task | Ansible debug | Re-run with `-vvv` flag |

## Problem: VirtualBox NAT IP (10.0.2.15)

**Symptom:**
```
error execution phase preflight: couldn't validate the identity of the API Server:
Get "https://10.0.2.15:6443/api/v1/namespaces/kube-public/configmaps/cluster-info":
dial tcp 10.0.2.15:6443: connect: connection refused
```

**Cause:** Kubespray detected VirtualBox NAT interface instead of host-only network.

**Fix:** Add explicit `ip=` to inventory:
```ini
k8s-ctr ansible_host=192.168.10.10 ip=192.168.10.10
```

**If already deployed with wrong IP:** Must reset and redeploy:
```bash
ansible-playbook -i inventory/mycluster/inventory.ini reset.yml -b
# Fix inventory, then:
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b
```

## Problem: etcd Health Check Failure

**Symptom:**
```
TASK [etcd : Configure | Wait for etcd cluster to be healthy]
fatal: [controller-0]: FAILED! => {"cmd": "etcdctl endpoint health"...
"dial tcp 192.168.10.100:2379: connect: connection refused"
```

**Diagnose:**
```bash
# On etcd node
systemctl status etcd
journalctl -u etcd -f

# Check if listening
ss -tlnp | grep 2379
```

**Common causes:**
1. **Wrong IP in etcd config** - Reset and redeploy with correct `ip=`
2. **Certificate mismatch** - Check `/etc/ssl/etcd/ssl/` permissions
3. **Firewall blocking** - Ensure ports 2379/2380 open

**Fix for stale state:**
```bash
ansible-playbook -i inventory/mycluster/inventory.ini reset.yml -b
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b
```

## Problem: Nodes Stuck NotReady

**Symptom:** `kubectl get nodes` shows NotReady status

**Diagnose:**
```bash
# Check kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Check CNI
ls /etc/cni/net.d/
ls /opt/cni/bin/

# Check node conditions
kubectl describe node <node-name>
```

**Common causes:**
1. **CNI not installed** - Check network_plugin role completed
2. **containerd not running** - `systemctl restart containerd`
3. **kubelet misconfigured** - Check `/etc/kubernetes/kubelet-config.yaml`

## Problem: "No hosts matched"

**Symptom:**
```
[WARNING]: Could not match supplied host pattern, ignoring: etcd
skipping: no hosts matched
```

**Cause:** Inventory path or syntax error

**Fix:**
```bash
# Use file path, not directory
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b

# Verify inventory parses correctly
ansible -i inventory/mycluster/inventory.ini etcd --list-hosts
ansible -i inventory/mycluster/inventory.ini kube_control_plane --list-hosts
```

## Problem: Container Runtime Not Running

**Symptom:**
```
[ERROR CRI]: container runtime is not running:
"transport: Error while dialing dial unix /var/run/containerd/containerd.sock:
connect: no such file or directory"
```

**Fix:**
```bash
# Check containerd
systemctl status containerd
journalctl -u containerd

# Restart if needed
systemctl restart containerd

# Verify socket exists
ls -la /var/run/containerd/containerd.sock
```

## Problem: Certificate Errors

**Symptom:**
```
x509: certificate has expired or is not yet valid
```

**Diagnose:**
```bash
# Check cert expiration
kubeadm certs check-expiration

# Check specific cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

**Fix:** See kubespray-certificates skill for renewal procedures.

## Reset Procedure

When deployment is corrupted beyond repair:

```bash
# Full reset - removes all K8s components
ansible-playbook -i inventory/mycluster/inventory.ini reset.yml -b

# Confirm with "yes" when prompted

# After reset, verify clean state
systemctl status kubelet  # should be inactive
ls /etc/kubernetes/        # should be empty/minimal

# Redeploy
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b
```

**Note:** Reset removes etcd data. All cluster state is lost.

## Log Locations

| Component | Log Command |
|-----------|-------------|
| etcd | `journalctl -u etcd` |
| kubelet | `journalctl -u kubelet` |
| containerd | `journalctl -u containerd` |
| API server | `kubectl logs -n kube-system kube-apiserver-<node>` |
| Ansible | Run with `-vvv` for debug output |

## Re-running After Failure

Ansible is idempotent - safe to re-run after fixing issues:

```bash
# Re-run full playbook (skips completed tasks)
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b

# Re-run specific tags only
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b --tags etcd
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b --tags network
```
