---
name: kubeadm-init
description: Use when initializing a Kubernetes control plane with kubeadm, setting up certificates, static pods, or troubleshooting init failures
---

# kubeadm init - Control Plane Initialization

## Overview

`kubeadm init` bootstraps the Kubernetes control plane through 14 automated phases.

**Core principle:** Understand each phase before troubleshooting. Certificate, network, and kubelet issues are traced to specific phases.

## When to Use

- Initializing a new Kubernetes cluster
- Troubleshooting control plane bootstrap failures
- Understanding certificate generation and static pod deployment
- Planning HA control plane setup

## The 14 Phases

```
kubeadm init phases (sequential):

1. preflight      → System requirements check
2. certs          → CA and component certificates
3. kubeconfig     → Component kubeconfig files
4. etcd           → Static Pod manifest for etcd
5. control-plane  → Static Pods for API server, controller-manager, scheduler
6. kubelet-start  → kubelet configuration and start
7. wait-control-plane → Wait for components to be healthy
8. upload-config  → Store config in ConfigMaps
9. upload-certs   → (--upload-certs) Store certs in Secret for HA
10. mark-control-plane → Add labels and taints
11. bootstrap-token → Create join token and RBAC
12. kubelet-finalize → Finalize kubelet TLS bootstrap
13. addon         → Install CoreDNS and kube-proxy
14. show-join-command → Display join command
```

## Quick Reference

| Phase | Output Location | Key Files |
|-------|-----------------|-----------|
| certs | `/etc/kubernetes/pki/` | ca.crt, apiserver.crt, etcd/ca.crt |
| kubeconfig | `/etc/kubernetes/` | admin.conf, kubelet.conf, scheduler.conf |
| etcd | `/etc/kubernetes/manifests/` | etcd.yaml |
| control-plane | `/etc/kubernetes/manifests/` | kube-apiserver.yaml, kube-scheduler.yaml |
| kubelet-start | `/var/lib/kubelet/` | config.yaml, kubeadm-flags.env |

## Configuration Approaches

### CLI Options (Simple)
```bash
kubeadm init \
  --apiserver-advertise-address=192.168.10.100 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/16 \
  --kubernetes-version=1.32.11
```

### Configuration File (Recommended)
```yaml
# kubeadm-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
- token: "123456.1234567890123456"
  ttl: "24h"
nodeRegistration:
  kubeletExtraArgs:
    node-ip: "192.168.10.100"
  criSocket: "unix:///run/containerd/containerd.sock"
localAPIEndpoint:
  advertiseAddress: "192.168.10.100"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "1.32.11"
networking:
  podSubnet: "10.244.0.0/16"      # Flannel default
  serviceSubnet: "10.96.0.0/16"
```

```bash
kubeadm init --config=kubeadm-init.yaml
```

## Critical Options

| Option | Purpose | When Required |
|--------|---------|---------------|
| `--apiserver-advertise-address` | API Server listen IP | Multi-NIC environments |
| `--control-plane-endpoint` | HA endpoint (LB address) | **Must set at init for future HA** |
| `--pod-network-cidr` | Pod IP range | Flannel: 10.244.0.0/16, Calico: 192.168.0.0/16 |
| `--apiserver-cert-extra-sans` | Additional cert SANs | Custom domains, external LB IPs |

## Phase-by-Phase Execution

Run individual phases when customization needed:

```bash
# Generate only certificates
kubeadm init phase certs all --config=kubeadm-init.yaml

# Generate manifests for customization
kubeadm init phase control-plane all --config=kubeadm-init.yaml
# Edit /etc/kubernetes/manifests/*.yaml
# Skip that phase in full init
kubeadm init --skip-phases=control-plane --config=kubeadm-init.yaml
```

## Pre-flight Verification

```bash
# Check required images
kubeadm config images list

# Pre-pull images (reduces init time)
kubeadm config images pull

# Dry-run to see what will happen
kubeadm init --config=kubeadm-init.yaml --dry-run
```

## Post-Init Steps

### 1. Configure kubectl
```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2. Install CNI (Required)
```bash
# Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Or Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 3. Verify
```bash
kubectl get nodes              # Should show Ready after CNI
kubectl get pods -n kube-system  # CoreDNS should become Running
```

## Common Issues

| Symptom | Phase | Cause | Fix |
|---------|-------|-------|-----|
| Port 6443 in use | preflight | Previous install | `kubeadm reset` first |
| CRI connection failed | preflight | containerd not running | `systemctl start containerd` |
| Node NotReady | addon | CNI not installed | Install Flannel/Calico |
| CoreDNS Pending | addon | CNI not installed | Install Flannel/Calico |
| Certificate SAN error | certs | Missing custom domain | Use `--apiserver-cert-extra-sans` |

## Certificate Structure

```
/etc/kubernetes/pki/
├── ca.crt, ca.key              # Cluster CA
├── apiserver.crt, apiserver.key
├── apiserver-kubelet-client.crt
├── front-proxy-ca.crt          # Aggregation Layer CA
├── front-proxy-client.crt
├── sa.key, sa.pub              # ServiceAccount signing
└── etcd/
    ├── ca.crt, ca.key          # etcd CA (separate from cluster CA)
    ├── server.crt, server.key
    ├── peer.crt, peer.key
    └── healthcheck-client.crt
```

## HA Considerations

**Must set at first init (cannot add later):**
- `--control-plane-endpoint`: Points to load balancer
- Can be DNS name initially pointing to single node

```bash
kubeadm init \
  --control-plane-endpoint "k8s-api.example.com:6443" \
  --upload-certs
```

## Reset and Retry

```bash
# Full reset
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd ~/.kube
iptables -F && iptables -t nat -F

# Then re-init
kubeadm init --config=kubeadm-init.yaml
```

## Verification Checklist

After successful init:
- [ ] `kubectl cluster-info` shows API server
- [ ] Static pods running: `crictl ps | grep -E 'apiserver|etcd|scheduler|controller'`
- [ ] Certificates in `/etc/kubernetes/pki/`
- [ ] kubeconfig files in `/etc/kubernetes/`
- [ ] kubelet running: `systemctl status kubelet`
- [ ] Join command displayed with token and CA hash
