---
name: kubespray-lab-setup
description: Use when setting up a local Vagrant/VirtualBox lab environment for Kubespray, provisioning multi-node clusters with Rocky Linux, or configuring admin/load balancer nodes for HA testing.
---

# Kubespray Lab Environment Setup

## Overview

This skill covers building a local 6-node Vagrant/VirtualBox lab for testing Kubespray HA deployments. The environment includes an admin/load balancer node, three control plane nodes with stacked etcd, and two worker nodes running Rocky Linux 10.

**Core principle:** The admin-lb node acts as your deployment workstation, HAProxy load balancer, and NFS server. All Kubespray operations are executed from this node against the five K8s nodes over a host-only network.

## When to Use

- Setting up a local Vagrant/VirtualBox lab for Kubespray testing
- Provisioning multi-node Rocky Linux clusters
- Configuring an admin node with HAProxy, NFS, and deployment tools
- Preparing K8s nodes with kernel modules and sysctl settings
- Testing HA configurations with a local load balancer

**Not for:** Running Kubespray playbooks (use kubespray-deployment), configuring HA variables (use kubespray-ha-configuration), troubleshooting cluster issues (use kubespray-troubleshooting)

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    Host Machine (macOS/Linux)                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │           VirtualBox Host-Only Network 192.168.10.0/24   │    │
│  │                                                          │    │
│  │  ┌──────────────┐                                        │    │
│  │  │  admin-lb     │  192.168.10.10                        │    │
│  │  │  HAProxy      │  API LB :6443 → CP nodes             │    │
│  │  │  NFS Server   │  Stats :9000  Prometheus :8405        │    │
│  │  │  Kubespray    │  NFS /srv/nfs/share                   │    │
│  │  └──────────────┘                                        │    │
│  │                                                          │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐         │    │
│  │  │ k8s-node1  │  │ k8s-node2  │  │ k8s-node3  │         │    │
│  │  │ CP + etcd  │  │ CP + etcd  │  │ CP + etcd  │         │    │
│  │  │ .10.11     │  │ .10.12     │  │ .10.13     │         │    │
│  │  └────────────┘  └────────────┘  └────────────┘         │    │
│  │                                                          │    │
│  │  ┌────────────┐  ┌────────────┐                          │    │
│  │  │ k8s-node4  │  │ k8s-node5  │                          │    │
│  │  │ Worker     │  │ Worker     │                          │    │
│  │  │ .10.14     │  │ .10.15     │                          │    │
│  │  └────────────┘  └────────────┘                          │    │
│  │                                                          │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Each VM: enp0s3 (NAT, internet) + enp0s8/enp0s9 (Host-Only)   │
└──────────────────────────────────────────────────────────────────┘
```

## VM Specifications

| Node | IP | Role | CPU | Memory |
|------|-----|------|-----|--------|
| admin-lb | 192.168.10.10 | HAProxy, NFS, Kubespray deployer | 2 | 1024 MB |
| k8s-node1 | 192.168.10.11 | Control Plane + etcd | 4 | 2048 MB |
| k8s-node2 | 192.168.10.12 | Control Plane + etcd | 4 | 2048 MB |
| k8s-node3 | 192.168.10.13 | Control Plane + etcd | 4 | 2048 MB |
| k8s-node4 | 192.168.10.14 | Worker | 4 | 2048 MB |
| k8s-node5 | 192.168.10.15 | Worker | 4 | 2048 MB |

**Network interfaces:**
- `enp0s3` - NAT adapter for internet access (VirtualBox default)
- `enp0s8` / `enp0s9` - Host-Only adapter for cluster communication (192.168.10.0/24)

## Vagrantfile

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX_IMAGE = "bento/rockylinux-10"
N = 5

Vagrant.configure("2") do |config|

  # --- k8s-node1 through k8s-node5 ---
  (1..N).each do |i|
    config.vm.define "k8s-node#{i}" do |node|
      node.vm.box = BOX_IMAGE
      node.vm.hostname = "k8s-node#{i}"
      node.vm.network "private_network", ip: "192.168.10.#{10 + i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "k8s-node#{i}"
        vb.memory = 2048
        vb.cpus = 4
        vb.linked_clone = true
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      end

      node.vm.provision "shell", path: "init_cfg.sh"
    end
  end

  # --- admin-lb node ---
  config.vm.define "admin-lb" do |admin|
    admin.vm.box = BOX_IMAGE
    admin.vm.hostname = "admin-lb"
    admin.vm.network "private_network", ip: "192.168.10.10"

    admin.vm.provider "virtualbox" do |vb|
      vb.name = "admin-lb"
      vb.memory = 1024
      vb.cpus = 2
      vb.linked_clone = true
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    end

    admin.vm.provision "shell", path: "admin-lb.sh"
  end
end
```

**Key settings:**
- `linked_clone = true` - saves disk space by sharing base image
- `nicpromisc2 allow-all` - enables promiscuous mode on the second NIC for CNI traffic (Calico/Flannel)
- K8s nodes are provisioned first so they are ready when admin-lb runs SSH key distribution

## Admin-LB Bootstrap Script (admin-lb.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==============================="
echo " admin-lb bootstrap"
echo "==============================="

# ---- Timezone ----
timedatectl set-timezone Asia/Seoul

# ---- Disable Firewall & SELinux ----
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# ---- SSH Configuration ----
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
echo 'root:qwe123' | chpasswd
systemctl restart sshd

# ---- Local DNS (/etc/hosts) ----
cat >> /etc/hosts <<EOF
192.168.10.10 admin-lb
192.168.10.11 k8s-node1
192.168.10.12 k8s-node2
192.168.10.13 k8s-node3
192.168.10.14 k8s-node4
192.168.10.15 k8s-node5
EOF

# ---- HAProxy ----
dnf install -y haproxy

cat > /etc/haproxy/haproxy.cfg <<'HAPCFG'
global
    log         127.0.0.1 local2
    maxconn     4096
    daemon

defaults
    mode    tcp
    log     global
    option  tcplog
    option  dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend k8s-api
    bind *:6443
    default_backend k8s-api-backend

backend k8s-api-backend
    balance roundrobin
    option tcp-check
    server k8s-node1 192.168.10.11:6443 check fall 3 rise 2
    server k8s-node2 192.168.10.12:6443 check fall 3 rise 2
    server k8s-node3 192.168.10.13:6443 check fall 3 rise 2

frontend stats
    bind *:9000
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST

frontend prometheus
    bind *:8405
    mode http
    http-request use-service prometheus-exporter if { path /metrics }
    no log
HAPCFG

setsebool -P haproxy_connect_any 1 2>/dev/null || true
systemctl enable --now haproxy

# ---- NFS Server ----
dnf install -y nfs-utils
mkdir -p /srv/nfs/share
cat >> /etc/exports <<EOF
/srv/nfs/share *(rw,async,no_root_squash)
EOF
systemctl enable --now nfs-server
exportfs -arv

# ---- kubectl ----
cat > /etc/yum.repos.d/kubernetes.repo <<'REPO'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
REPO
dnf install -y kubectl
kubectl completion bash > /etc/bash_completion.d/kubectl

# ---- k9s ----
curl -Lo /tmp/k9s.rpm https://github.com/derailed/k9s/releases/latest/download/k9s_linux_amd64.rpm
rpm -ivh /tmp/k9s.rpm
rm -f /tmp/k9s.rpm

# ---- Helm ----
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | \
  DESIRED_VERSION=v3.16.2 bash
helm completion bash > /etc/bash_completion.d/helm

# ---- SSH Key Distribution ----
dnf install -y sshpass
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa -q
for i in 1 2 3 4 5; do
  sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.10.1${i}
done

# ---- Kubespray ----
dnf install -y git python3-pip
git clone -b v2.29.1 https://github.com/kubernetes-sigs/kubespray.git /root/kubespray
pip3 install --break-system-packages -r /root/kubespray/requirements.txt

echo "==============================="
echo " admin-lb bootstrap complete"
echo "==============================="
```

## Node Init Script (init_cfg.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==============================="
echo " K8s node init: $(hostname)"
echo "==============================="

# ---- Timezone ----
timedatectl set-timezone Asia/Seoul

# ---- Disable Swap ----
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# ---- Disable Firewall & SELinux ----
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# ---- Kernel Modules for Kubernetes ----
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# ---- Sysctl Settings ----
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# ---- Local DNS (/etc/hosts) ----
cat >> /etc/hosts <<EOF
192.168.10.10 admin-lb
192.168.10.11 k8s-node1
192.168.10.12 k8s-node2
192.168.10.13 k8s-node3
192.168.10.14 k8s-node4
192.168.10.15 k8s-node5
EOF

# ---- SSH Configuration ----
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
echo 'root:qwe123' | chpasswd
systemctl restart sshd

echo "==============================="
echo " K8s node init complete: $(hostname)"
echo "==============================="
```

## Deploying the Environment

### 1. Start All VMs

```bash
vagrant up
```

This takes 15-30 minutes. K8s nodes provision first (init_cfg.sh), then admin-lb provisions last (admin-lb.sh including SSH key distribution).

### 2. Verify VM Status

```bash
vagrant status
# Expected: all 6 VMs running
```

### 3. Connect to Admin Node

```bash
vagrant ssh admin-lb
```

### 4. Test Connectivity to All Nodes

```bash
for i in 1 2 3 4 5; do
  ssh root@192.168.10.1${i} hostname
done
# Should print k8s-node1 through k8s-node5
```

### 5. Verify HAProxy

```bash
# Check service status
systemctl status haproxy

# Stats page (from host browser)
# http://192.168.10.10:9000/stats

# Prometheus metrics
curl http://192.168.10.10:8405/metrics

# API frontend will show DOWN until K8s is deployed (expected)
```

### 6. Verify NFS

```bash
showmount -e localhost
# Expected: /srv/nfs/share *
```

## Rocky Linux 10 Considerations

### Python 3.12 and PEP 668

Rocky Linux 10 ships Python 3.12, which enforces PEP 668 (externally managed environments). Direct `pip install` fails:

```
error: externally-managed-environment
```

**Fix:** Use the `--break-system-packages` flag:

```bash
pip3 install --break-system-packages -r requirements.txt
```

This is already included in the admin-lb.sh script. Alternatively, use a virtual environment:

```bash
python3 -m venv /root/kubespray-venv
source /root/kubespray-venv/bin/activate
pip install -r /root/kubespray/requirements.txt
```

## Common Errors (Searchable)

```
VBoxManage: error: Could not find a controller named 'SATA Controller'
```
**Cause:** VirtualBox storage controller mismatch with box image. **Fix:** Destroy the VM (`vagrant destroy <name>`) and re-create. If persistent, update VirtualBox.

```
The IP address configured for the host-only network is not within the allowed ranges
```
**Cause:** VirtualBox 6.1.28+ restricts host-only network ranges. **Fix:** Create or edit `/etc/vbox/networks.conf`:
```
* 192.168.10.0/24
```

```
Timed out while waiting for the machine to boot
```
**Cause:** Insufficient host resources or VirtualBox issue. **Fix:** Reduce VM count or memory, check `VBoxManage list runningvms`.

```
sshpass: ssh-copy-id failed (exit code 6)
```
**Cause:** Target node SSH not ready when admin-lb provisions. **Fix:** Re-run provisioning: `vagrant provision admin-lb`. The Vagrantfile defines K8s nodes first to minimize this.

```
error: externally-managed-environment
```
**Cause:** Python 3.12 PEP 668 on Rocky Linux 10. **Fix:** Add `--break-system-packages` to pip install.

```
E: Failed to connect to HAProxy stats socket
```
**Cause:** HAProxy not running or misconfigured. **Fix:** Check `systemctl status haproxy` and validate `/etc/haproxy/haproxy.cfg` syntax with `haproxy -c -f /etc/haproxy/haproxy.cfg`.

```
mount.nfs: access denied by server
```
**Cause:** NFS exports not applied. **Fix:** Run `exportfs -arv` and verify with `showmount -e localhost`.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Forgetting `--nicpromisc2 allow-all` | CNI (Calico/Flannel) traffic dropped between VMs |
| Not setting `ip=` in Kubespray inventory | Kubespray detects 10.0.2.15 (NAT) instead of 192.168.10.x |
| Starting admin-lb before K8s nodes | SSH key distribution fails because nodes are not ready |
| Missing `/etc/vbox/networks.conf` | VirtualBox rejects 192.168.10.0/24 host-only range |
| Skipping `swapoff -a` | kubelet refuses to start with swap enabled |
| Not disabling SELinux | Container runtime and CNI operations fail |
| Using `pip install` without `--break-system-packages` | PEP 668 blocks installation on Rocky Linux 10 |
| Forgetting `linked_clone = true` | Each VM copies the full base image (2-4 GB each instead of ~200 MB) |
| Wrong Kubespray branch vs K8s version | Version mismatch causes deployment failure |
