---
name: kubeadm-troubleshooting
description: Use when kubeadm init fails, join fails, nodes show NotReady, pods stuck Pending, certificate errors, or kubelet crashlooping
---

# kubeadm Troubleshooting

## Overview

Systematic approach to diagnosing kubeadm cluster issues.

**Core principle:** Trace symptoms to specific components. Most issues are: certificates, networking, or kubelet configuration.

## When to Use

- `kubeadm init` or `kubeadm join` fails
- Nodes show `NotReady` status
- Pods stuck in `Pending` state
- Certificate-related errors
- kubelet crashlooping
- Control plane components failing

## Diagnostic Flowchart

```
Symptom
   │
   ├─ kubeadm init fails ────────────▶ Check pre-flight errors
   │                                   Check port conflicts
   │                                   Check containerd status
   │
   ├─ kubeadm join fails ────────────▶ Check token validity
   │                                   Check network connectivity
   │                                   Check firewall (port 6443)
   │
   ├─ Node NotReady ─────────────────▶ Check CNI installation
   │                                   Check kubelet logs
   │                                   Check node conditions
   │
   ├─ Pod Pending ───────────────────▶ If CoreDNS: install CNI
   │                                   If other: check node resources
   │                                   Check taints/tolerations
   │
   ├─ Certificate error ─────────────▶ Check certificate expiry
   │                                   Check SAN configuration
   │                                   Check clock synchronization
   │
   └─ kubelet crashloop ─────────────▶ Check config.yaml exists
                                       Check containerd running
                                       Check cgroup driver match
```

## Quick Diagnostic Commands

```bash
# Cluster overview
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info

# Node details
kubectl describe node <node-name>

# Component health
kubectl get componentstatuses  # deprecated but sometimes useful
kubectl get --raw='/readyz?verbose'

# kubelet status
systemctl status kubelet
journalctl -u kubelet -f --no-pager

# containerd status
systemctl status containerd
crictl ps
crictl pods
crictl info

# Network
ss -tlnp | grep -E '6443|10250|2379'
ip route
```

## Issue: kubeadm init Fails

### Pre-flight Errors

```bash
# See what's failing
kubeadm init --dry-run 2>&1 | grep -E '\[ERROR\]|\[WARNING\]'
```

| Error | Fix |
|-------|-----|
| `[ERROR CRI]: container runtime not running` | `systemctl start containerd` |
| `[ERROR Swap]: running with swap on` | `swapoff -a` |
| `[ERROR Port-6443]: Port 6443 in use` | Previous cluster: `kubeadm reset` |
| `[ERROR DirAvailable]: /etc/kubernetes not empty` | `rm -rf /etc/kubernetes/*` |
| `[ERROR FileAvailable--etc-kubernetes-manifests-*]` | Clean previous manifests |

### Port Conflicts

```bash
# Check what's using required ports
ss -tlnp | grep -E '6443|2379|2380|10250|10259|10257'

# If previous cluster
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd
iptables -F && iptables -t nat -F
```

### containerd Issues

```bash
# Check containerd running
systemctl status containerd

# Check CRI plugin enabled (should NOT be in disabled list)
grep disabled_plugins /etc/containerd/config.toml

# Check socket
ls -la /run/containerd/containerd.sock

# Test with crictl
crictl info
```

## Issue: kubeadm join Fails

### Connection Refused

```bash
# From worker node - test API server reachability
curl -k https://192.168.10.100:6443/healthz
# Expected: "ok"
# If "Connection refused": firewall or API not listening

# On control plane - check API listening
ss -tlnp | grep 6443
# Should show: *:6443 or 0.0.0.0:6443

# Check firewall
systemctl is-active firewalld
# If active:
firewall-cmd --list-ports
# Or disable:
systemctl disable --now firewalld
```

### Token Expired

```bash
# On control plane - check token validity
kubeadm token list
# If empty or expired:
kubeadm token create --print-join-command
```

### CA Hash Mismatch

```bash
# Regenerate correct hash on control plane
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```

### TLS Bootstrap Timeout

```bash
# On worker - check kubelet logs during join
journalctl -u kubelet -f

# Common causes:
# - Firewall blocking 6443
# - Wrong advertise address (multi-NIC)
# - Network routing issues
```

## Issue: Node NotReady

### Check Node Conditions

```bash
kubectl describe node <node-name> | grep -A 20 Conditions
```

| Condition | False Means |
|-----------|-------------|
| Ready | Node unhealthy |
| MemoryPressure | Memory OK |
| DiskPressure | Disk OK |
| PIDPressure | PIDs OK |
| NetworkUnavailable | CNI working |

### NetworkUnavailable = True (CNI Missing)

```bash
# Check if CNI installed
ls /etc/cni/net.d/
# Empty = no CNI

# Install Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Verify
kubectl get pods -n kube-flannel
kubectl get nodes  # Should become Ready
```

### kubelet Not Reporting

```bash
# On the NotReady node
systemctl status kubelet
journalctl -u kubelet --no-pager | tail -50

# Common issues:
# - config.yaml missing: kubeadm join not completed
# - containerd socket missing: containerd not running
# - cgroup driver mismatch: SystemdCgroup not set
```

## Issue: Pods Stuck Pending

### CoreDNS Pending (No CNI)

```bash
kubectl get pods -n kube-system | grep coredns
# coredns-xxx   0/1   Pending

kubectl describe pod -n kube-system coredns-xxx
# Events: FailedScheduling - no nodes available to schedule

# Fix: Install CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Other Pods Pending

```bash
kubectl describe pod <pod-name> -n <namespace>
# Check Events section

# Common causes:
# - Insufficient resources
# - Node taints without tolerations
# - Node selector mismatch
# - PVC not bound
```

### Check Taints (Control Plane Only Cluster)

```bash
# Control plane has NoSchedule taint by default
kubectl describe node | grep Taints

# To schedule on control plane (single-node cluster)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Issue: Certificate Errors

### Check Certificate Expiry

```bash
# All certificates
kubeadm certs check-expiration

# Specific certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### Certificate SAN Issues

```bash
# Check API server certificate SANs
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 "Subject Alternative Name"

# If missing required SAN, regenerate:
kubeadm init phase certs apiserver --apiserver-cert-extra-sans=new.domain.com
```

### Renew Certificates

```bash
# Renew all
kubeadm certs renew all

# Restart control plane components
crictl pods --name='kube-apiserver|kube-controller|kube-scheduler|etcd' -q | xargs -I {} crictl stop {}
```

### Clock Skew

```bash
# Check time on all nodes
date

# Enable NTP
timedatectl set-ntp true
chronyc sources -v
```

## Issue: kubelet Crashlooping

### Before kubeadm init/join (Expected)

```bash
journalctl -u kubelet | grep "config.yaml"
# "failed to load kubelet config file, path: /var/lib/kubelet/config.yaml"

# This is NORMAL before kubeadm init or join
# kubelet needs config from kubeadm
```

### After kubeadm init/join

```bash
# Check config exists
ls -la /var/lib/kubelet/config.yaml

# Check kubeconfig exists
ls -la /etc/kubernetes/kubelet.conf

# Check containerd
systemctl status containerd

# Check cgroup driver match
grep cgroupDriver /var/lib/kubelet/config.yaml
# Should be: cgroupDriver: systemd

grep SystemdCgroup /etc/containerd/config.toml
# Should be: SystemdCgroup = true
```

## Issue: etcd Problems

### Check etcd Health

```bash
# Using etcdctl with certs
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health

# Check etcd logs
crictl logs $(crictl ps -q --name=etcd)
```

### etcd Data Corruption (Last Resort)

```bash
# WARNING: Destroys cluster state
kubeadm reset -f
rm -rf /var/lib/etcd
kubeadm init ...
```

## Full Reset Procedure

```bash
# On ALL nodes
kubeadm reset -f

# Clean directories
rm -rf /etc/kubernetes
rm -rf /var/lib/kubelet
rm -rf /var/lib/etcd
rm -rf ~/.kube

# Clean iptables
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

# Clean CNI
rm -rf /etc/cni/net.d
rm -rf /var/lib/cni

# Restart containerd
systemctl restart containerd

# Then re-init on control plane
kubeadm init --config=kubeadm-init.yaml
```

## Log Collection Script

```bash
#!/bin/bash
LOGDIR="/tmp/k8s-debug-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$LOGDIR"

echo "Collecting system info..."
hostnamectl > "$LOGDIR/hostnamectl.txt"
ip addr > "$LOGDIR/ip-addr.txt"
ip route > "$LOGDIR/ip-route.txt"
ss -tlnp > "$LOGDIR/ss-tlnp.txt"

echo "Collecting service status..."
systemctl status kubelet > "$LOGDIR/kubelet-status.txt" 2>&1
systemctl status containerd > "$LOGDIR/containerd-status.txt" 2>&1

echo "Collecting logs..."
journalctl -u kubelet --no-pager > "$LOGDIR/kubelet.log" 2>&1
journalctl -u containerd --no-pager > "$LOGDIR/containerd.log" 2>&1

echo "Collecting Kubernetes info..."
kubectl get nodes -o wide > "$LOGDIR/nodes.txt" 2>&1
kubectl get pods -A -o wide > "$LOGDIR/pods.txt" 2>&1
kubectl describe nodes > "$LOGDIR/nodes-describe.txt" 2>&1

echo "Collecting directory listings..."
ls -la /etc/kubernetes/ > "$LOGDIR/etc-kubernetes.txt" 2>&1
ls -la /var/lib/kubelet/ > "$LOGDIR/var-lib-kubelet.txt" 2>&1
ls -la /etc/cni/net.d/ > "$LOGDIR/cni-config.txt" 2>&1

echo "Collecting crictl info..."
crictl info > "$LOGDIR/crictl-info.txt" 2>&1
crictl ps -a > "$LOGDIR/crictl-ps.txt" 2>&1
crictl pods > "$LOGDIR/crictl-pods.txt" 2>&1

tar -czf "$LOGDIR.tar.gz" -C /tmp "$(basename $LOGDIR)"
echo "Logs collected: $LOGDIR.tar.gz"
```

## Quick Reference: Component Locations

| Component | Config | Logs |
|-----------|--------|------|
| kubelet | `/var/lib/kubelet/config.yaml` | `journalctl -u kubelet` |
| containerd | `/etc/containerd/config.toml` | `journalctl -u containerd` |
| API Server | `/etc/kubernetes/manifests/kube-apiserver.yaml` | `crictl logs <id>` |
| etcd | `/etc/kubernetes/manifests/etcd.yaml` | `crictl logs <id>` |
| Certificates | `/etc/kubernetes/pki/` | N/A |
| kubeconfig | `/etc/kubernetes/*.conf` | N/A |
| CNI | `/etc/cni/net.d/` | Varies by CNI |
