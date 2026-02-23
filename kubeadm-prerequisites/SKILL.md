---
name: kubeadm-prerequisites
description: Use when preparing nodes for kubeadm installation, configuring containerd, kernel modules, swap, SELinux, or troubleshooting pre-flight check failures
---

# kubeadm Prerequisites

## Overview

kubeadm requires specific system configuration before cluster initialization.

**Core principle:** kubeadm does NOT provision machines. It expects containerd, kubelet, and kernel settings already configured.

## When to Use

- Preparing new nodes for Kubernetes
- Troubleshooting pre-flight check failures
- Setting up containerd with SystemdCgroup
- Configuring kernel modules and parameters

## Pre-flight Requirements Summary

| Requirement | Check | Fix |
|-------------|-------|-----|
| Swap disabled | `free -h \| grep Swap` | `swapoff -a` + remove from fstab |
| containerd running | `systemctl status containerd` | Install and configure |
| Required ports free | `ss -tlnp \| grep 6443` | Stop conflicting service |
| 2+ CPUs | `nproc` | Upgrade hardware |
| 2GB+ RAM | `free -h` | Upgrade hardware |
| Unique hostname | `hostname` | Change if duplicate |
| br_netfilter loaded | `lsmod \| grep br_netfilter` | `modprobe br_netfilter` |

## System Configuration

### Disable Swap (Required)

```bash
# Immediate disable
swapoff -a

# Permanent (remove swap from fstab)
sed -i '/swap/d' /etc/fstab

# Verify
free -h | grep Swap
# Swap:             0B          0B          0B
```

**Why?** Kubernetes scheduler cannot accurately predict memory if swap is used. OOM handling becomes unpredictable.

### SELinux Configuration

```bash
# Check current status
getenforce
# Enforcing

# Set to Permissive
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Permissive
```

**Why Permissive?** Enforcing mode may block container filesystem access. Permissive logs violations without blocking.

### Disable Firewall (Lab) or Open Ports (Production)

```bash
# Lab environment - disable completely
systemctl disable --now firewalld

# Production - open required ports
# Control Plane
firewall-cmd --permanent --add-port=6443/tcp   # API Server
firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
firewall-cmd --permanent --add-port=10250/tcp  # kubelet
firewall-cmd --permanent --add-port=10259/tcp  # scheduler
firewall-cmd --permanent --add-port=10257/tcp  # controller-manager
firewall-cmd --reload

# Worker Nodes
firewall-cmd --permanent --add-port=10250/tcp  # kubelet
firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort
firewall-cmd --reload
```

### Time Synchronization

```bash
# Enable NTP
timedatectl set-ntp true

# Set timezone
timedatectl set-timezone Asia/Seoul

# Verify
chronyc sources -v
chronyc tracking
```

**Why?** Certificate validity depends on accurate time. Clock skew causes TLS errors.

## Kernel Configuration

### Required Modules

```bash
# Load modules
modprobe overlay
modprobe br_netfilter

# Verify loaded
lsmod | grep -E 'overlay|br_netfilter'

# Persist across reboot
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

| Module | Purpose |
|--------|---------|
| overlay | Container image layer filesystem |
| br_netfilter | Bridge traffic through iptables |

### Required Parameters

```bash
# Set parameters
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply
sysctl --system

# Verify
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

| Parameter | Purpose |
|-----------|---------|
| bridge-nf-call-iptables | Route bridge traffic through iptables (Pod networking) |
| ip_forward | Enable IP forwarding (node as router) |

## containerd Installation

### Add Docker Repository (RHEL/Rocky)

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf makecache
```

### Install containerd

```bash
# Check available versions
dnf list --showduplicates containerd.io

# Install specific version
dnf install -y containerd.io-2.1.5-1.el10

# Verify
containerd --version
runc --version
```

### Configure containerd

```bash
# Generate default config
containerd config default | tee /etc/containerd/config.toml

# Enable SystemdCgroup (CRITICAL)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Verify
grep SystemdCgroup /etc/containerd/config.toml
# SystemdCgroup = true
```

**Why SystemdCgroup?** kubelet and containerd must use same cgroup driver. systemd is recommended for systems using systemd init.

### Start containerd

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd

# Verify socket
ls -l /run/containerd/containerd.sock
```

## kubeadm/kubelet/kubectl Installation

### Add Kubernetes Repository

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

dnf makecache
```

### Install Components

```bash
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable kubelet (will crashloop until kubeadm init/join)
systemctl enable --now kubelet

# Verify versions
kubeadm version
kubelet --version
kubectl version --client
```

### Configure crictl

```bash
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF

# Verify
crictl info
```

## hosts File Configuration

```bash
# Remove problematic localhost mappings (Vagrant adds these)
sed -i '/^127\.0\.\(1\|2\)\.1/d' /etc/hosts

# Add cluster nodes
cat <<EOF >> /etc/hosts
192.168.10.100 k8s-ctr
192.168.10.101 k8s-w1
192.168.10.102 k8s-w2
EOF

# Verify
ping -c 1 k8s-ctr
```

## Version Compatibility

### containerd vs Kubernetes

| Kubernetes | containerd |
|------------|------------|
| 1.32 | 2.1.0+, 2.0.1+, 1.7.24+ |
| 1.33 | 2.1.0+, 2.0.4+, 1.7.24+ |
| 1.34 | 2.1.3+, 2.0.6+, 1.7.28+ |
| 1.35 | 2.2.0+, 2.1.5+ |

### config.toml Version

| containerd | config.toml version |
|------------|---------------------|
| 1.x | version = 2 |
| 2.x | version = 3 |

## Common Pre-flight Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `[ERROR CRI]` | containerd not running | `systemctl start containerd` |
| `[ERROR Swap]` | Swap enabled | `swapoff -a` |
| `[ERROR Port-6443]` | Port in use | Previous cluster not cleaned |
| `[ERROR NumCPU]` | < 2 CPUs | Upgrade VM/hardware |
| `[ERROR Mem]` | < 1700MB RAM | Upgrade VM/hardware |
| `[ERROR FileContent]` | Kernel params not set | Apply sysctl settings |
| `[ERROR SystemVerification]` | br_netfilter not loaded | `modprobe br_netfilter` |

## Verification Script

```bash
#!/bin/bash
echo "=== Kubernetes Prerequisites Check ==="

echo -n "Swap: "
[[ $(swapon -s | wc -l) -eq 0 ]] && echo "OK (disabled)" || echo "FAIL (enabled)"

echo -n "SELinux: "
[[ $(getenforce) != "Enforcing" ]] && echo "OK ($(getenforce))" || echo "WARN (Enforcing)"

echo -n "Firewall: "
systemctl is-active firewalld &>/dev/null && echo "WARN (active)" || echo "OK (inactive)"

echo -n "containerd: "
systemctl is-active containerd &>/dev/null && echo "OK (running)" || echo "FAIL (not running)"

echo -n "br_netfilter: "
lsmod | grep -q br_netfilter && echo "OK (loaded)" || echo "FAIL (not loaded)"

echo -n "ip_forward: "
[[ $(sysctl -n net.ipv4.ip_forward) -eq 1 ]] && echo "OK (enabled)" || echo "FAIL (disabled)"

echo -n "SystemdCgroup: "
grep -q "SystemdCgroup = true" /etc/containerd/config.toml 2>/dev/null && echo "OK" || echo "FAIL"

echo -n "kubelet: "
which kubelet &>/dev/null && echo "OK ($(kubelet --version 2>&1 | awk '{print $2}'))" || echo "FAIL (not installed)"
```

## Quick Setup (All Steps)

```bash
# As root on each node

# 1. System config
swapoff -a && sed -i '/swap/d' /etc/fstab
setenforce 0 && sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
systemctl disable --now firewalld
timedatectl set-ntp true

# 2. Kernel modules
modprobe overlay && modprobe br_netfilter
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 3. Kernel parameters
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 4. containerd
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y containerd.io
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl enable --now containerd

# 5. Kubernetes
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet

# 6. crictl
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF
```

**Note:** kubelet will crashloop until `kubeadm init` or `kubeadm join` is executed. This is expected.
