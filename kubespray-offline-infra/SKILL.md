---
name: kubespray-offline-infra
description: Use when setting up air-gap infrastructure services for Kubernetes offline deployment - DNS servers (bind), NTP time sync (chrony), NAT gateways, local YUM/DNF package repositories (reposync), or private PyPI mirrors (devpi) in isolated networks.
---

# Air-Gap Infrastructure Services for Kubernetes Offline Deployment

## Overview

Air-gap Kubernetes deployment requires several infrastructure services running on the admin server before cluster deployment can begin. These services replace what would normally be provided by the internet: DNS resolution, time synchronization, network routing, package repositories, and Python package mirrors.

**Core principle:** The admin server acts as the central hub providing all network services to the isolated internal network. Every node in the cluster depends on the admin server for name resolution, time sync, package installation, and container image pulls.

## When to Use

- Setting up a DNS server (bind) for name resolution in isolated networks
- Configuring NTP time synchronization (chrony) in air-gap environments
- Creating a NAT gateway for controlled or temporary internet access
- Building local YUM/DNF package mirrors via reposync
- Setting up a PyPI mirror (devpi or pypi-mirror) for Python packages
- Preparing the admin server before running kubespray-offline

## Network Architecture

```
Internet <-> Admin Server (192.168.10.10) <-> Internal Network (192.168.10.0/24)
                |                                    |
          DNS, NTP, NAT GW                    k8s-node1 (192.168.10.11)
          Registry, Repos                     k8s-node2 (192.168.10.12)
```

The admin server has two network interfaces:
- External-facing interface (e.g., enp0s8) with internet access
- Internal-facing interface (e.g., enp0s9) on the 192.168.10.0/24 subnet

All k8s nodes reside on the internal network and route through the admin server.

## NAT Gateway Setup

The NAT gateway allows internal nodes to reach the internet through the admin server. This is useful during initial setup and can be disabled later to enforce full air-gap isolation.

### On the admin server

```bash
# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1
cat <<EOF > /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward = 1
EOF
sysctl --system

# Configure NAT masquerading on the external interface
iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE

# To remove NAT and isolate the network (for testing air-gap)
iptables -t nat -D POSTROUTING -o enp0s8 -j MASQUERADE
```

### On k8s nodes -- route through admin

```bash
# Disable the external interface on nodes
nmcli connection down enp0s8
nmcli connection modify enp0s8 connection.autoconnect no

# Add default route through the admin server on the internal interface
nmcli connection modify enp0s9 +ipv4.routes "0.0.0.0/0 192.168.10.10 200"
nmcli connection up enp0s9
```

Verify connectivity from a node:

```bash
ping -c 2 192.168.10.10
curl -s http://192.168.10.10 > /dev/null && echo "Admin reachable"
```

## DNS Server (bind)

### On the admin server

```bash
dnf install -y bind bind-utils
```

Edit `/etc/named.conf` with the following key settings:

```
options {
    listen-on port 53 { any; };
    allow-query { 127.0.0.1; 192.168.10.0/24; };
    allow-recursion { 127.0.0.1; 192.168.10.0/24; };
    forwarders { 168.126.63.1; 8.8.8.8; };
    recursion yes;
    dnssec-validation auto;
};
```

Start and enable the service:

```bash
systemctl enable --now named

# Point the admin server itself to its own DNS
echo "nameserver 192.168.10.10" > /etc/resolv.conf
```

### On k8s nodes

Disable NetworkManager DNS management so it does not overwrite `/etc/resolv.conf`:

```bash
cat <<EOF > /etc/NetworkManager/conf.d/99-dns-none.conf
[main]
dns=none
EOF
systemctl restart NetworkManager

# Point to admin server DNS
echo "nameserver 192.168.10.10" > /etc/resolv.conf
```

## NTP Server (chrony)

Time synchronization is critical for Kubernetes certificate validation and log consistency.

### On the admin server

```bash
cat <<EOF > /etc/chrony.conf
server pool.ntp.org iburst
server kr.pool.ntp.org iburst
allow 192.168.10.0/24
local stratum 10
logdir /var/log/chrony
EOF
systemctl restart chronyd
```

Verify:

```bash
chronyc sources -v
chronyc clients
```

### On k8s nodes

```bash
cat <<EOF > /etc/chrony.conf
server 192.168.10.10 iburst
logdir /var/log/chrony
EOF
systemctl restart chronyd
timedatectl status
```

## Local YUM/DNF Repository

Syncing repositories takes 12+ minutes for a full mirror. Plan accordingly.

### On the admin server

```bash
dnf install -y dnf-plugins-core createrepo nginx
mkdir -p /data/repos/rocky/10

# Sync repositories
dnf reposync --repoid=baseos --download-metadata -p /data/repos/rocky/10     # ~6GB, ~3min
dnf reposync --repoid=appstream --download-metadata -p /data/repos/rocky/10  # ~14GB, ~9min
dnf reposync --repoid=extras --download-metadata -p /data/repos/rocky/10     # ~67MB, fast
```

Configure nginx to serve the repository:

```bash
cat <<EOF > /etc/nginx/conf.d/repos.conf
server {
    listen 80;
    server_name repo-server;
    location /rocky/10/ {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        root /data/repos;
    }
}
EOF
systemctl enable --now nginx
```

### On k8s nodes

Back up existing repos and point to the internal mirror:

```bash
mkdir -p /etc/yum.repos.d/backup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/

cat <<EOF > /etc/yum.repos.d/internal-rocky.repo
[internal-baseos]
name=Internal Rocky 10 BaseOS
baseurl=http://192.168.10.10/rocky/10/baseos
enabled=1
gpgcheck=0

[internal-appstream]
name=Internal Rocky 10 AppStream
baseurl=http://192.168.10.10/rocky/10/appstream
enabled=1
gpgcheck=0

[internal-extras]
name=Internal Rocky 10 Extras
baseurl=http://192.168.10.10/rocky/10/extras
enabled=1
gpgcheck=0
EOF

dnf clean all && dnf repolist && dnf makecache
```

## Private PyPI Mirror

Two approaches are available depending on requirements.

### Approach A: devpi-server (interactive, supports caching)

Best for environments where you want a caching proxy that can also host uploaded packages.

```bash
pip install devpi-server devpi-client devpi-web
devpi-init --serverdir /data/devpi_data
nohup devpi-server --serverdir /data/devpi_data --host 0.0.0.0 --port 3141 > /var/log/devpi.log 2>&1 &

# Upload packages
devpi use http://ADMIN_IP:3141
devpi login root --password ""
devpi index -c prod bases=root/pypi
devpi use root/prod
pip download jmespath netaddr -d /tmp/pypi-packages
devpi upload /tmp/pypi-packages/*
```

### Approach B: pypi-mirror (static, used by kubespray-offline)

Best for fully static mirrors served by nginx. This is the approach used by the kubespray-offline project.

```bash
pip install python-pypi-mirror
pip download -d outputs/pypi/files -r requirements.txt
pypi-mirror create -d outputs/pypi/files -m outputs/pypi
# Served by nginx on port 80
```

### Client configuration (all nodes)

```bash
cat <<EOF > /etc/pip.conf
[global]
index-url = http://ADMIN_IP/pypi
trusted-host = ADMIN_IP
timeout = 60
EOF
```

Replace `ADMIN_IP` with the actual admin server IP (e.g., 192.168.10.10).

## Quick Reference Table

| Service | Port | Software | Admin Config | Node Config |
|---------|------|----------|-------------|-------------|
| DNS | 53 | bind | /etc/named.conf | /etc/resolv.conf |
| NTP | 123 | chrony | /etc/chrony.conf (server) | /etc/chrony.conf (client) |
| NAT | - | iptables | MASQUERADE rule | default route via admin |
| YUM/DNF | 80 | nginx + reposync | /etc/nginx/conf.d/ | /etc/yum.repos.d/ |
| PyPI | 3141 or 80 | devpi or pypi-mirror | devpi-init | /etc/pip.conf |

## Common Mistakes

- **Forgetting to disable SELinux before bind setup.** SELinux can block bind from listening on port 53 or reading zone files. Either configure SELinux policies or set it to permissive during setup.
- **Not disabling NetworkManager DNS management on nodes.** NetworkManager will overwrite `/etc/resolv.conf` on every network event, undoing manual DNS configuration.
- **Missing `allow 192.168.10.0/24` in chrony config.** Without this directive, chrony will reject NTP requests from the internal network.
- **Not running `createrepo` after syncing packages.** If reposync is run without `--download-metadata`, you must generate repository metadata manually with `createrepo`.
- **pypi-mirror vs devpi: the `+simple` endpoint is required for pip.** When using devpi, the index URL must include the `/+simple/` suffix. When using pypi-mirror, ensure the directory structure matches what pip expects.

## Verification Commands

```bash
# DNS -- verify name resolution through the admin server
dig +short google.com @192.168.10.10

# NTP -- check time synchronization status
chronyc sources -v
timedatectl status

# YUM/DNF repo -- confirm internal repos are active
dnf repolist
dnf makecache

# PyPI -- test package installation from internal mirror
pip install --dry-run somepackage
```

## Recommended Setup Order

1. **NAT Gateway** -- enables temporary internet access for downloading packages
2. **DNS Server** -- all subsequent services benefit from name resolution
3. **NTP Server** -- time sync is a prerequisite for TLS and certificates
4. **YUM/DNF Repository** -- mirrors OS packages for offline node provisioning
5. **PyPI Mirror** -- mirrors Python packages needed by kubespray and tools
6. **Disable NAT** -- once all mirrors are populated, remove the NAT rule to enforce air-gap
