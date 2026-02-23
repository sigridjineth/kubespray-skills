---
name: rke2-deployment
description: Use when deploying Kubernetes clusters with RKE2 (Rancher Kubernetes Engine 2), configuring server and agent nodes, managing built-in Helm chart addons, or setting up CIS-hardened clusters. Use when seeing "rke2-server failed to start" or "unable to join cluster" errors.
---

# RKE2 Deployment

## Overview

RKE2 (also known as RKE Government) is Rancher's next-generation Kubernetes distribution focused on security and compliance. It deploys a fully conformant Kubernetes cluster using a single binary that manages containerd, kubelet, kube-proxy, and control plane components as static pods. Unlike Kubespray (which orchestrates kubeadm via Ansible), RKE2 is a self-contained installer that handles everything from container runtime to CNI in a single process.

**Core principle:** RKE2 is security-first -- it ships with CIS Benchmark compliance, FIPS 140-2 support via BoringCrypto, and SELinux policies out of the box. Everything runs on containerd with minimal host OS dependencies.

## When to Use

- Deploying new Kubernetes clusters with RKE2
- Setting up RKE2 server (control plane) and agent (worker) nodes
- Configuring built-in Helm chart addons (Canal, CoreDNS, metrics-server, ingress-nginx)
- Customizing CNI and addon behavior via HelmChartConfig manifests
- Building CIS-hardened or FIPS-compliant clusters

**Not for:** Kubespray-based deployments (use kubespray-deployment), RKE2 upgrades and day-2 operations (use rke2-operations), air-gapped RKE2 installations (see RKE2 documentation for tarball-based offline install)

## Quick Reference

| File / Path | Purpose |
|-------------|---------|
| `/etc/rancher/rke2/config.yaml` | Server and agent configuration |
| `/etc/rancher/rke2/rke2.yaml` | Generated kubeconfig (server only) |
| `/var/lib/rancher/rke2/bin/` | RKE2 binaries (kubectl, crictl, containerd, ctr, runc) |
| `/var/lib/rancher/rke2/server/manifests/` | Static pod and HelmChartConfig manifests |
| `/var/lib/rancher/rke2/server/node-token` | Join token for agents and additional servers |
| `/var/lib/rancher/rke2/agent/etc/containerd/` | Containerd configuration and registry mirrors |
| `/usr/bin/rke2-uninstall.sh` | Server uninstall script |
| `/usr/bin/rke2-agent-uninstall.sh` | Agent uninstall script |

## Architecture

### Server + Agent Model

```
┌─────────────────────────────────────────────────────────┐
│                   RKE2 Server Node                       │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              rke2-server (supervisor)              │  │
│  │                                                   │  │
│  │  ┌─────────────┐  ┌──────────────────────────┐   │  │
│  │  │ containerd   │  │  Static Pod Manifests     │   │  │
│  │  │  + kubelet   │  │  /var/lib/rancher/rke2/  │   │  │
│  │  │  + kube-proxy│  │  server/manifests/        │   │  │
│  │  └──────┬──────┘  └────────────┬─────────────┘   │  │
│  │         │                      │                  │  │
│  │         ▼                      ▼                  │  │
│  │  ┌──────────────────────────────────────────┐    │  │
│  │  │         Control Plane Pods                │    │  │
│  │  │  kube-apiserver    kube-scheduler         │    │  │
│  │  │  kube-controller-manager    etcd          │    │  │
│  │  └──────────────────────────────────────────┘    │  │
│  │                                                   │  │
│  │  ┌──────────────────────────────────────────┐    │  │
│  │  │         Built-in Helm Charts              │    │  │
│  │  │  rke2-canal       rke2-coredns            │    │  │
│  │  │  rke2-metrics-server  rke2-ingress-nginx  │    │  │
│  │  │  rke2-runtimeclasses                      │    │  │
│  │  └──────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Listens: :6443 (API)  :9345 (supervisor/join)          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   RKE2 Agent Node                        │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              rke2-agent (supervisor)               │  │
│  │                                                   │  │
│  │  ┌─────────────┐  ┌──────────────────────────┐   │  │
│  │  │ containerd   │  │   Workload Pods           │   │  │
│  │  │  + kubelet   │  │   (scheduled by           │   │  │
│  │  │  + kube-proxy│  │    control plane)          │   │  │
│  │  └─────────────┘  └──────────────────────────┘   │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Connects to server :9345 for join and supervision      │
└─────────────────────────────────────────────────────────┘
```

### Boot Sequence

```
RKE2 Server Start
      │
      ▼
1. rke2-server process starts (supervisor)
      │
      ▼
2. containerd starts as managed subprocess
      │
      ▼
3. kubelet starts, watches static pod manifest directory
   /var/lib/rancher/rke2/server/manifests/
      │
      ▼
4. Static pod manifests written for control plane:
   etcd, kube-apiserver, kube-controller-manager, kube-scheduler
      │
      ▼
5. Control plane pods launched by kubelet
      │
      ▼
6. Built-in Helm charts deployed:
   rke2-canal, rke2-coredns, rke2-metrics-server, rke2-ingress-nginx
      │
      ▼
7. Supervisor API available on :9345
   Kubernetes API available on :6443
      │
      ▼
   Server Ready (~2 minutes)
```

### Security-First Design

| Feature | Description |
|---------|-------------|
| CIS Benchmark | Ships with CIS Kubernetes Benchmark compliance by default |
| BoringCrypto | FIPS 140-2 validated cryptographic module |
| SELinux | Includes rke2-selinux RPM for proper SELinux policies |
| GPG-signed packages | RPM repos are GPG-signed for supply chain security |
| Pod Security Standards | Enforces restricted pod security by default in CIS mode |
| Secrets encryption | Supports encryption-at-rest for Kubernetes secrets |

### Key Design Choices vs kubeadm/Kubespray

| Aspect | RKE2 | kubeadm/Kubespray |
|--------|------|-------------------|
| Control plane taints | No taints by default (workloads can run on CP) | Taints CP nodes by default |
| Addon management | All addons managed as built-in Helm charts | Ansible roles deploy addons |
| Container runtime | Embedded containerd (managed by RKE2 process) | Separately installed containerd |
| Package signing | GPG-signed RPM repos | Apt/Yum repos with standard signing |
| CNI default | Canal (Calico + Flannel) | Configurable (Calico, Flannel, Cilium) |
| Configuration | Single config.yaml per node | Ansible group_vars across many files |

## Installation

### Download and Install (Online)

```bash
# Download the installer script
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh

# Install RKE2 server (default type)
INSTALL_RKE2_CHANNEL=v1.33 ./install.sh

# Or install in one line
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.33 sh -
```

### What the Installer Does

The installer installs three RPM packages:

| Package | Purpose |
|---------|---------|
| `rke2-common` | Core RKE2 binaries and systemd units |
| `rke2-selinux` | SELinux policy module for RKE2 |
| `rke2-server` | Server-specific configuration (or `rke2-agent` for workers) |

### Installed Binaries

After installation, binaries are placed at `/var/lib/rancher/rke2/bin/`:

```bash
ls /var/lib/rancher/rke2/bin/
# kubectl  crictl  containerd  ctr  runc
```

### Create System-Wide Symlinks

```bash
# Make RKE2 binaries available system-wide
ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
ln -sf /var/lib/rancher/rke2/bin/crictl /usr/local/bin/crictl

# Or add to PATH in profile
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> /etc/profile.d/rke2.sh
source /etc/profile.d/rke2.sh
```

## Server Configuration

### /etc/rancher/rke2/config.yaml

Create the configuration directory and file before starting the server:

```bash
mkdir -p /etc/rancher/rke2
```

#### Minimal Server Config

```yaml
# /etc/rancher/rke2/config.yaml (server)
write-kubeconfig-mode: "0644"
cni: canal
```

#### Full Server Config Example

```yaml
# /etc/rancher/rke2/config.yaml (server)

# --- Kubeconfig ---
write-kubeconfig-mode: "0644"

# --- Debugging ---
debug: false

# --- CNI Plugin ---
# Options: canal (default), cilium, calico, none
cni: canal

# --- Network Binding ---
bind-address: 0.0.0.0
advertise-address: 192.168.56.11
node-ip: 192.168.56.11

# --- Disable Components ---
disable-cloud-controller: true
disable:
  - rke2-ingress-nginx          # Disable built-in ingress if using custom
  - rke2-coredns-autoscaler     # Disable if managing scaling manually
  # - servicelb                 # Disable ServiceLB if using MetalLB
```

### CNI Options

| CNI | Value | Notes |
|-----|-------|-------|
| Canal | `canal` | Default. Calico network policy + Flannel overlay. Best general-purpose choice. |
| Cilium | `cilium` | eBPF-based. Advanced observability and network policy. Higher resource usage. |
| Calico | `calico` | Full Calico with BGP support. Good for on-prem without overlay. |
| None | `none` | Bring your own CNI. Install manually after cluster bootstrap. |

### HelmChartConfig: Customizing Built-in Addons

RKE2 manages all addons as Helm charts. To override default Helm values, create HelmChartConfig resources in the server manifests directory.

**Important:** Place HelmChartConfig files in `/var/lib/rancher/rke2/server/manifests/` **before** starting `rke2-server` for the first time. If the server is already running, the changes are picked up automatically.

#### Example: Customize Canal Flannel Interface

```yaml
# /var/lib/rancher/rke2/server/manifests/rke2-canal-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    flannel:
      iface: enp0s8
```

This is critical in multi-NIC environments (e.g., VirtualBox with NAT + host-only). Without specifying the interface, Canal/Flannel may pick the NAT interface and pod networking breaks.

#### Example: Customize CoreDNS Autoscaler

```yaml
# /var/lib/rancher/rke2/server/manifests/rke2-coredns-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-coredns
  namespace: kube-system
spec:
  valuesContent: |-
    autoscaler:
      enabled: true
      min: 2
      max: 10
      coresPerReplica: 128
      nodesPerReplica: 8
```

#### Example: Customize Ingress NGINX

```yaml
# /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      publishService:
        enabled: true
      config:
        use-forwarded-headers: "true"
```

## Starting the Server

### Enable and Start

```bash
# Enable and start RKE2 server (takes ~2 minutes)
systemctl enable --now rke2-server.service
```

### Monitor Startup

```bash
# Watch startup logs in real time
journalctl -u rke2-server -f

# Key log lines to watch for:
# "Running kube-apiserver"     -- API server starting
# "Running kube-scheduler"     -- Scheduler starting
# "Running etcd"               -- etcd starting
# "Node password validated"    -- Node registered successfully
# "Tunnel server egress proxy" -- Agent tunnel ready
```

Startup takes approximately 2 minutes. The server is ready when you see the tunnel server egress proxy message and no error loops.

### Configure kubectl Access

```bash
# Copy kubeconfig to standard location
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config

# Verify access
kubectl get nodes
```

If accessing from a remote machine, copy the kubeconfig and replace the server address:

```bash
# On the remote machine
scp root@<server-ip>:/etc/rancher/rke2/rke2.yaml ~/.kube/config
sed -i 's/127.0.0.1/<server-ip>/g' ~/.kube/config

# macOS variant
sed -i '' 's/127.0.0.1/<server-ip>/g' ~/.kube/config
```

### Built-in Helm Charts

After the server starts, these Helm charts are deployed automatically:

| Chart | Purpose |
|-------|---------|
| `rke2-canal` | CNI plugin (Canal = Calico policy + Flannel overlay) |
| `rke2-coredns` | Cluster DNS |
| `rke2-metrics-server` | Resource metrics for `kubectl top` and HPA |
| `rke2-ingress-nginx` | Ingress controller (unless disabled) |
| `rke2-runtimeclasses` | RuntimeClass definitions for containerd |

Verify with:

```bash
helm list -A
```

## Joining Agent (Worker) Nodes

### Step 1: Get the Join Token from the Server

```bash
# On the server node
cat /var/lib/rancher/rke2/server/node-token
```

The token is a long string like `K10...::server:...`. This is used by both agents and additional servers to join the cluster.

### Step 2: Install RKE2 Agent on Worker Node

```bash
# Install RKE2 as agent type
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_CHANNEL=v1.33 sh -
```

### Step 3: Configure the Agent

```bash
mkdir -p /etc/rancher/rke2
```

```yaml
# /etc/rancher/rke2/config.yaml (agent)
server: https://192.168.56.11:9345
token: <paste-token-from-server>
node-ip: 192.168.56.12
```

**Important:** The agent connects to port **9345** (supervisor port), NOT 6443 (API port). The supervisor port handles node registration, certificate issuance, and kubelet bootstrapping.

### Step 4: Start the Agent

```bash
systemctl enable --now rke2-agent.service
```

### Step 5: Monitor Agent Join

```bash
# On the agent node
journalctl -u rke2-agent -f

# On the server node, verify the new node appears
kubectl get nodes
```

### Joining Additional Server Nodes (HA)

For HA control plane, additional server nodes join the first server:

```yaml
# /etc/rancher/rke2/config.yaml (additional server)
server: https://192.168.56.11:9345
token: <paste-token-from-server>
write-kubeconfig-mode: "0644"
cni: canal
advertise-address: 192.168.56.13
node-ip: 192.168.56.13
```

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.33 sh -
systemctl enable --now rke2-server.service
```

## Lab Setup with Vagrant

### 2-Node Lab: Server + Agent

This minimal lab creates one RKE2 server (control plane) and one RKE2 agent (worker) on Rocky Linux 9.6.

```
┌──────────────────────────────────────────────────────┐
│               Host Machine (macOS/Linux)              │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │     VirtualBox Host-Only Network              │    │
│  │     192.168.56.0/24                           │    │
│  │                                               │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  │    │
│  │  │    k8s-node1     │  │    k8s-node2     │  │    │
│  │  │    RKE2 Server   │  │    RKE2 Agent    │  │    │
│  │  │    (CP + etcd)   │  │    (Worker)      │  │    │
│  │  │  192.168.56.11   │  │  192.168.56.12   │  │    │
│  │  │                  │  │                  │  │    │
│  │  │  :6443 API       │  │  Connects to     │  │    │
│  │  │  :9345 Supervisor│  │  :9345 to join   │  │    │
│  │  └──────────────────┘  └──────────────────┘  │    │
│  │                                               │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  Each VM: enp0s3 (NAT) + enp0s8 (Host-Only)         │
└──────────────────────────────────────────────────────┘
```

| Node | IP | Role | CPU | Memory |
|------|-----|------|-----|--------|
| k8s-node1 | 192.168.56.11 | RKE2 Server (Control Plane + etcd) | 4 | 4096 MB |
| k8s-node2 | 192.168.56.12 | RKE2 Agent (Worker) | 4 | 4096 MB |

### Vagrant Commands

```bash
# Start all VMs
vagrant up

# SSH into server node
vagrant ssh k8s-node1

# SSH into agent node
vagrant ssh k8s-node2

# Destroy lab environment
vagrant destroy -f
```

### Deployment Procedure on Lab

#### On k8s-node1 (Server)

```bash
# 1. Install RKE2 server
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
INSTALL_RKE2_CHANNEL=v1.33 ./install.sh

# 2. Create configuration
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml <<EOF
write-kubeconfig-mode: "0644"
cni: canal
advertise-address: 192.168.56.11
node-ip: 192.168.56.11
EOF

# 3. (Optional) Customize Canal for multi-NIC
mkdir -p /var/lib/rancher/rke2/server/manifests
cat > /var/lib/rancher/rke2/server/manifests/rke2-canal-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    flannel:
      iface: enp0s8
EOF

# 4. Start server (~2 min)
systemctl enable --now rke2-server.service

# 5. Monitor startup
journalctl -u rke2-server -f

# 6. Set up kubectl
ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

# 7. Get join token for agent
cat /var/lib/rancher/rke2/server/node-token
```

#### On k8s-node2 (Agent)

```bash
# 1. Install RKE2 agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_CHANNEL=v1.33 sh -

# 2. Create configuration (use token from server)
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://192.168.56.11:9345
token: <paste-token-from-server>
node-ip: 192.168.56.12
EOF

# 3. Start agent
systemctl enable --now rke2-agent.service

# 4. Monitor join
journalctl -u rke2-agent -f
```

## Post-Deployment Verification

### 1. Cluster Info and Node Status

```bash
kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:6443
# CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/...

kubectl get nodes -o wide
# NAME        STATUS   ROLES                       AGE   VERSION
# k8s-node1   Ready    control-plane,etcd,master   5m    v1.33.x+rke2r1
# k8s-node2   Ready    <none>                      3m    v1.33.x+rke2r1
```

### 2. System Pod Health Check

```bash
kubectl get pods -A -o wide
```

Expected pods in a healthy cluster:

| Pod | Namespace | Expected Count | Runs On |
|-----|-----------|----------------|---------|
| etcd | kube-system | 1 per server node | Server nodes (static pod) |
| kube-apiserver | kube-system | 1 per server node | Server nodes (static pod) |
| kube-controller-manager | kube-system | 1 per server node | Server nodes (static pod) |
| kube-scheduler | kube-system | 1 per server node | Server nodes (static pod) |
| kube-proxy | kube-system | 1 per node (DaemonSet) | All nodes |
| canal (calico + flannel) | kube-system | 1 per node (DaemonSet) | All nodes |
| rke2-coredns | kube-system | 2 replicas (default) | Any schedulable node |
| rke2-metrics-server | kube-system | 1 replica | Any schedulable node |
| rke2-ingress-nginx-controller | kube-system | 1 per node (DaemonSet) | All nodes (unless disabled) |

All pods should show `Running` status with `READY` columns showing full readiness (e.g., `1/1`).

### 3. Helm Chart Verification

```bash
helm list -A
# NAME                    NAMESPACE       STATUS      CHART
# rke2-canal              kube-system     deployed    rke2-canal-v3.x.x
# rke2-coredns            kube-system     deployed    rke2-coredns-1.x.x
# rke2-metrics-server     kube-system     deployed    rke2-metrics-server-3.x.x
# rke2-ingress-nginx      kube-system     deployed    rke2-ingress-nginx-4.x.x
# rke2-runtimeclasses     kube-system     deployed    rke2-runtimeclasses-0.x.x
```

### 4. CNI (Canal) Verification

```bash
# Verify Canal pods are running on all nodes
kubectl get pods -n kube-system -l k8s-app=canal -o wide

# Check that each node has a flannel interface
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'
```

### 5. CoreDNS Verification

```bash
# Test DNS resolution inside the cluster
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local

# Expected:
# Server:    10.43.0.10
# Address:   10.43.0.1
```

### 6. Metrics Server

```bash
# May take 1-2 minutes after deployment for first scrape
kubectl top nodes
kubectl top pods -A
```

### 7. API Server Direct Test

```bash
# Test API server endpoint directly
curl -sk https://192.168.56.11:6443/version
# Should return JSON with gitVersion matching deployed version
```

## Uninstall

### Server Uninstall

```bash
# Stop the service first
systemctl stop rke2-server.service
systemctl disable rke2-server.service

# Run the uninstall script
/usr/bin/rke2-uninstall.sh
```

This removes:
- RKE2 binaries and RPM packages
- containerd state and images
- etcd data directory
- Kubernetes configuration and certificates
- CNI configuration and state
- systemd unit files

### Agent Uninstall

```bash
# Stop the service first
systemctl stop rke2-agent.service
systemctl disable rke2-agent.service

# Run the agent uninstall script
/usr/bin/rke2-agent-uninstall.sh
```

### Post-Uninstall Cleanup

```bash
# Verify no RKE2 processes remain
ps aux | grep rke2

# Remove any leftover directories
rm -rf /etc/rancher/rke2
rm -rf /var/lib/rancher/rke2
rm -rf /var/lib/kubelet
# Remove symlinks
rm -f /usr/local/bin/kubectl
rm -f /usr/local/bin/crictl
```

## Common Errors (Searchable)

```
rke2-server.service: Failed with result 'exit-code'
```
**Cause:** Server process failed to start. Could be port conflict, bad config, or insufficient resources. **Fix:** Check `journalctl -u rke2-server -xe` for the specific error. Ensure ports 6443 and 9345 are free. Verify `/etc/rancher/rke2/config.yaml` is valid YAML.

```
level=fatal msg="starting kubernetes: preparing server: bootstrap data already found and target server is not ready"
```
**Cause:** A previous RKE2 installation left behind state. **Fix:** Run `/usr/bin/rke2-uninstall.sh` to fully clean up, then reinstall.

```
msg="unable to join cluster" error="Node password rejected"
```
**Cause:** Agent node-token does not match the server's expected token. **Fix:** Re-read the token from `/var/lib/rancher/rke2/server/node-token` on the server and update the agent config.

```
level=error msg="Failed to connect to proxy" error="dial tcp <ip>:9345: connect: connection refused"
```
**Cause:** Agent cannot reach the server supervisor port. **Fix:** Verify the server is running (`systemctl status rke2-server`), check firewall rules allow port 9345, and confirm the `server:` URL in agent config is correct.

```
E0101 00:00:00.000000 kubelet.go:2855] "Container runtime network not ready" networkReady="NetworkReady=false"
```
**Cause:** CNI plugin has not started yet. **Fix:** Wait 1-2 minutes for Canal/Cilium to initialize. If it persists, check CNI pod logs: `kubectl logs -n kube-system -l k8s-app=canal`.

```
error: error loading config file "/etc/rancher/rke2/rke2.yaml": open /etc/rancher/rke2/rke2.yaml: permission denied
```
**Cause:** kubeconfig file has restrictive permissions. **Fix:** Set `write-kubeconfig-mode: "0644"` in config.yaml, or run `chmod 644 /etc/rancher/rke2/rke2.yaml`, or use `sudo` with kubectl.

```
level=fatal msg="starting kubernetes: preparing server: etcd: etcdserver: mvcc: database space exceeded"
```
**Cause:** etcd database has run out of space (default 2 GB quota). **Fix:** Compact and defragment etcd. For emergency recovery, increase `--quota-backend-bytes` in etcd args.

```
unable to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox
```
**Cause:** CNI plugin is misconfigured or crashed. **Fix:** Check Canal/Calico pod logs in kube-system namespace. In multi-NIC environments, verify the HelmChartConfig sets the correct flannel interface.

```
x509: certificate signed by unknown authority
```
**Cause:** kubectl is using an old or wrong kubeconfig. **Fix:** Re-copy kubeconfig from `/etc/rancher/rke2/rke2.yaml` and ensure you are pointing to the correct cluster.

```
rke2-agent.service: Start request repeated too quickly
```
**Cause:** Agent is crash-looping on startup. **Fix:** Check `journalctl -u rke2-agent -xe` for the root cause. Common causes: wrong token, unreachable server, or port blocked by firewall.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Agent connects to port 6443 instead of 9345 | Use `server: https://<ip>:9345` in agent config (9345 is the supervisor port) |
| Starting server without `config.yaml` | Create `/etc/rancher/rke2/config.yaml` before first start |
| Forgetting `write-kubeconfig-mode: "0644"` | Add to config.yaml or `chmod` the kubeconfig after generation |
| Not specifying flannel interface in multi-NIC | Create HelmChartConfig for rke2-canal with `flannel.iface` set |
| HelmChartConfig placed in wrong directory | Must be in `/var/lib/rancher/rke2/server/manifests/`, not `/etc/rancher/` |
| Expecting control plane node taints | RKE2 does NOT taint control plane nodes by default (unlike kubeadm) |
| Using wrong token for agent join | Re-read token from `/var/lib/rancher/rke2/server/node-token` on the server |
| Not waiting for server startup (~2 min) | Monitor `journalctl -u rke2-server -f` and wait for supervisor ready |
| Running `kubectl` without PATH or symlink | RKE2 kubectl is at `/var/lib/rancher/rke2/bin/kubectl` -- create symlink to `/usr/local/bin/` |
| Uninstalling without stopping service first | Stop the service with `systemctl stop rke2-server` before running uninstall script |
| Expecting `/etc/kubernetes/admin.conf` | RKE2 kubeconfig is at `/etc/rancher/rke2/rke2.yaml`, not the kubeadm path |
| Old state from previous install causes failures | Run `/usr/bin/rke2-uninstall.sh` for clean slate before reinstalling |
