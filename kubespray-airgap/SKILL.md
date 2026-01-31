---
name: kubespray-airgap
description: Use when deploying Kubernetes in air-gapped or offline environments, setting up private container registries, staging binaries for offline use, or configuring containerd registry mirrors.
---

# Kubespray Air-Gap Deployment

## Overview

Air-gapped environments (banks, government, defense) have no internet access. Kubespray requires all binaries and container images to be staged internally before deployment.

**Core principle:** Stage everything (binaries + images) inside the network before running Kubespray. Configure mirrors so containerd fetches from internal sources.

## When to Use

- Deploying to networks with no internet access
- Setting up private container registries
- Staging Kubernetes binaries internally
- Configuring containerd to use internal mirrors

**Not for:** Online deployments (use kubespray-deployment), troubleshooting (use kubespray-troubleshooting)

## What You Need Inside the Air-Gap

1. **Private container registry** - Harbor, Nexus, or any OCI-compliant registry
2. **HTTP file server** - Hosts binaries (containerd, runc, etcd, kubeadm, etc.)
3. **Ansible control node** - Inside the isolated network

## Identifying Required Files

### Generate Download List

```bash
# On internet-connected machine
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --tags download \
  -e download_run_once=true \
  -e download_localhost=true
```

Files download to `/tmp/releases/` (or `local_release_dir`).

### Required Binaries

```
/var/www/files/
├── kubernetes/v1.32.0/{kubeadm,kubectl,kubelet}
├── containerd/v2.0.0/containerd-2.0.0-linux-amd64.tar.gz
├── runc/v1.2.0/runc.amd64
├── cni-plugins/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz
├── etcd/v3.5.15/etcd-v3.5.15-linux-amd64.tar.gz
├── crictl/v1.31.0/crictl-v1.31.0-linux-amd64.tar.gz
└── nerdctl/v2.0.0/nerdctl-2.0.0-linux-amd64.tar.gz
```

### Required Container Images

**Note:** Version numbers are examples. Check `roles/download/defaults/main/main.yml` for current versions.

```
registry.k8s.io/pause:3.x
registry.k8s.io/coredns/coredns:v1.x.x
registry.k8s.io/kube-proxy:v1.x.x
registry.k8s.io/metrics-server/metrics-server:v0.x.x
quay.io/coreos/flannel:v0.x.x   # if using Flannel
docker.io/calico/...:v3.x.x     # if using Calico
```

### Generate Complete Image List

```bash
# List all images Kubespray will use for your configuration
cd kubespray

# Method 1: Parse download defaults
grep -r "image:" roles/download/defaults/ | grep -oP '(?<=: ).*' | sort -u

# Method 2: Use contrib script (if available)
./contrib/offline/generate_list.sh

# Method 3: Dry-run and capture image references
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --tags download -e download_run_once=true --check -v 2>&1 | \
  grep -oP '[\w./]+:[\w.-]+' | sort -u
```

## Staging Binaries

### Configure Kubespray for Internal File Server

Create `inventory/mycluster/group_vars/all/offline.yml`:

```yaml
# Internal file server
files_repo: "http://files.internal.example.com"

# Kubernetes binaries
kubeadm_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubelet"

# Container runtime
containerd_download_url: "{{ files_repo }}/containerd/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
runc_download_url: "{{ files_repo }}/runc/v{{ runc_version }}/runc.{{ image_arch }}"
crictl_download_url: "{{ files_repo }}/crictl/v{{ crictl_version }}/crictl-v{{ crictl_version }}-linux-{{ image_arch }}.tar.gz"
nerdctl_download_url: "{{ files_repo }}/nerdctl/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-linux-{{ image_arch }}.tar.gz"

# CNI and etcd
cni_download_url: "{{ files_repo }}/cni-plugins/v{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-v{{ cni_version }}.tgz"
etcd_download_url: "{{ files_repo }}/etcd/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
```

## Populating Private Registry

### Pull, Tag, Push Script

```bash
#!/bin/bash
PRIVATE_REGISTRY="registry.internal.example.com"

IMAGES=(
  "registry.k8s.io/pause:3.10"
  "registry.k8s.io/coredns/coredns:v1.11.3"
  "registry.k8s.io/kube-proxy:v1.32.0"
  "quay.io/coreos/flannel:v0.26.1"
)

for IMAGE in "${IMAGES[@]}"; do
  # Extract image name without registry
  NAME=$(echo $IMAGE | sed 's|.*/||')

  docker pull $IMAGE
  docker tag $IMAGE ${PRIVATE_REGISTRY}/${NAME}
  docker push ${PRIVATE_REGISTRY}/${NAME}
done
```

Run on internet-connected machine, then transfer registry data to air-gap.

## Configuring containerd Mirrors

Create `inventory/mycluster/group_vars/all/containerd.yml`:

```yaml
containerd_registries_mirrors:
  - prefix: registry.k8s.io
    mirrors:
      - host: https://registry.internal.example.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
        ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.crt

  - prefix: docker.io
    mirrors:
      - host: https://registry.internal.example.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
        ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.crt

  - prefix: quay.io
    mirrors:
      - host: https://registry.internal.example.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
        ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.crt

  - prefix: ghcr.io
    mirrors:
      - host: https://registry.internal.example.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
        ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.crt
```

### Resulting Configuration on Nodes

```
/etc/containerd/certs.d/
├── registry.k8s.io/hosts.toml
├── docker.io/hosts.toml
├── quay.io/hosts.toml
└── ghcr.io/hosts.toml
```

## Override Image Repositories

```yaml
# inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
kube_image_repo: "registry.internal.example.com"
gcr_image_repo: "registry.internal.example.com"
docker_image_repo: "registry.internal.example.com"
quay_image_repo: "registry.internal.example.com"

# Critical: sandbox (pause) image
pod_infra_image_repo: "registry.internal.example.com"
pod_infra_image_tag: "3.10"
```

## Package Manager Dependencies

Kubespray installs packages (conntrack, socat, etc.). Options:

1. **Internal repo mirror** (recommended)
   - Use reposync (RHEL) or apt-mirror (Debian)
   - Point nodes at internal mirror

2. **Pre-install packages**
   - Build golden image with all dependencies
   - Use for all Kubernetes nodes

3. **Skip package management** (risky)
   - Ensure all dependencies pre-installed

## Running Offline Deployment

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -e download_run_once=false \
  -e download_localhost=false \
  -b
```

Or if binaries are pre-staged on nodes:
```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -e skip_downloads=true \
  -b
```

## Verification

After deployment, verify no external connections:

```bash
# Check containerd logs for external registry access
journalctl -u containerd | grep -i "registry.k8s.io\|docker.io\|quay.io"

# Should see only internal registry
journalctl -u containerd | grep "registry.internal"
```

## Common Errors (Searchable)

```
Error response from daemon: Get "https://registry.internal.example.com/v2/": x509: certificate signed by unknown authority
```
**Fix:** Distribute CA cert to nodes, set `ca_file` in containerd config, run `update-ca-trust`

```
failed to pull image "registry.k8s.io/pause:3.10": failed to resolve reference
```
**Fix:** Image missing from private registry. Pull, tag, push the image.

```
TASK [download : download_container | Download image] failed
curl: (7) Failed to connect to files.internal.example.com port 80: Connection refused
```
**Fix:** Internal file server not reachable. Check DNS, firewall, service status.

```
sha256 checksum mismatch
```
**Fix:** Binary corrupted during transfer. Re-download and verify checksum.

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Missing one image | Deployment proceeds, DaemonSet fails later. Verify all images staged. |
| Self-signed cert errors | Distribute CA cert to nodes, configure `ca_file` |
| DNS resolution failure | Use IP addresses or configure internal DNS |
| Binary architecture mismatch | Verify amd64 vs arm64 matches target nodes |
| Version mismatch | Keep staged files synced with Kubespray version variables |

## Complete offline.yml Example

```yaml
# inventory/mycluster/group_vars/all/offline.yml

files_repo: "http://files.internal.example.com"

# Kubernetes
kubeadm_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubelet"

# Container runtime
containerd_download_url: "{{ files_repo }}/containerd/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
runc_download_url: "{{ files_repo }}/runc/v{{ runc_version }}/runc.{{ image_arch }}"
nerdctl_download_url: "{{ files_repo }}/nerdctl/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-linux-{{ image_arch }}.tar.gz"
crictl_download_url: "{{ files_repo }}/crictl/v{{ crictl_version }}/crictl-v{{ crictl_version }}-linux-{{ image_arch }}.tar.gz"

# CNI and etcd
cni_download_url: "{{ files_repo }}/cni-plugins/v{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-v{{ cni_version }}.tgz"
etcd_download_url: "{{ files_repo }}/etcd/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"

# Image registries (all point to internal)
kube_image_repo: "registry.internal.example.com"
gcr_image_repo: "registry.internal.example.com"
docker_image_repo: "registry.internal.example.com"
quay_image_repo: "registry.internal.example.com"
```
