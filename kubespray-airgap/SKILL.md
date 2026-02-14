---
name: kubespray-airgap
description: Use when deploying Kubernetes in air-gapped or offline environments using kubespray-offline tool, setting up private container registries, staging binaries and images for offline use, configuring containerd registry mirrors, or troubleshooting image pull failures in isolated networks.
---

# Kubespray Air-Gap Deployment with kubespray-offline

## Overview

Air-gapped environments have no internet access. The kubespray-offline tool automates staging all binaries and container images required for a fully offline Kubernetes deployment.

**Core principle:** kubespray-offline downloads everything on an internet-connected machine, produces a self-contained `outputs/` directory, then deploys entirely offline on the target network. No internet is needed after the transfer.

## When to Use

- Deploying Kubernetes to networks with no internet access
- Using the kubespray-offline tool for offline deployment
- Setting up private container registries for Kubernetes
- Configuring containerd to use internal registry mirrors
- Troubleshooting image pull issues in air-gapped environments

**Not for:** Online deployments (use kubespray-deployment), general troubleshooting (use kubespray-troubleshooting).

## kubespray-offline Tool Workflow

### download-all.sh Pipeline

The `download-all.sh` script runs these steps in order on an internet-connected machine:

```
config.sh --> precheck.sh --> prepare-pkgs.sh --> prepare-py.sh --> get-kubespray.sh -->
pypi-mirror.sh --> download-kubespray-files.sh --> download-additional-containers.sh -->
create-repo.sh --> copy-target-scripts.sh
```

### Configuration Variables (config.sh)

Set versions in `config.sh` (and `target-scripts/config.sh`):

```bash
KUBESPRAY_VERSION=2.30.0
RUNC_VERSION=1.3.4
CONTAINERD_VERSION=2.2.1
NERDCTL_VERSION=2.2.1
CNI_VERSION=1.8.0
NGINX_VERSION=1.29.4
REGISTRY_VERSION=3.0.0
REGISTRY_PORT=35000
```

## Outputs Directory Structure

After `download-all.sh` completes, the `outputs/` directory contains everything needed:

```
outputs/
├── config.sh                # Version configs
├── files/                   # Binaries (kubectl, kubelet, kubeadm, containerd, runc, etcd, cni, helm, etc.)
├── images/                  # Container images as .tar.gz + images.list + additional-images.list
├── pypi/                    # Python packages + index
├── rpms/                    # RPM packages + repo metadata
├── playbook/                # offline-repo ansible role
├── setup-all.sh             # Master setup script
├── setup-container.sh       # Install containerd + load nginx/registry images
├── install-containerd.sh    # Containerd installation from local files
├── start-nginx.sh           # Start nginx container (file server)
├── start-registry.sh        # Start registry container (port 35000)
├── setup-offline.sh         # Configure offline yum repo + pypi mirror
├── setup-py.sh              # Install python from offline repo
├── load-push-all-images.sh  # Load images to containerd + push to registry
├── extract-kubespray.sh     # Extract kubespray tarball + apply patches
└── nginx-default.conf       # Nginx config
```

Transfer this entire `outputs/` directory to the admin server inside the air-gapped network.

## Deployment Pipeline (Step by Step)

Run these steps on the admin server inside the air-gapped network:

```bash
# [1] Install containerd, load nginx/registry images
cd outputs && ./setup-container.sh

# [2] Start nginx (serves files, rpms, pypi on port 80)
./start-nginx.sh

# [3] Configure offline repos (yum + pypi mirror)
./setup-offline.sh

# [4] Install python from offline repo
./setup-py.sh

# [5] Start private registry (port 35000)
./start-registry.sh

# [6] Load all images and push to registry
./load-push-all-images.sh

# [7] Extract kubespray
./extract-kubespray.sh
```

**ARM64 note:** Use `--all-platforms` flag when loading images:

```bash
./load-push-all-images.sh --all-platforms
```

## offline.yml Configuration

Create `inventory/mycluster/group_vars/all/offline.yml` with all download URLs pointing to the admin server. Replace `ADMIN_IP` with the actual IP of the admin server.

```yaml
http_server: "http://ADMIN_IP"
registry_host: "ADMIN_IP:35000"

containerd_registries_mirrors:
  - prefix: "{{ registry_host }}"
    mirrors:
      - host: "http://{{ registry_host }}"
        capabilities: ["pull", "resolve"]
        skip_verify: true

files_repo: "{{ http_server }}/files"
yum_repo: "{{ http_server }}/rpms"

# All image repos point to private registry
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"
github_image_repo: "{{ registry_host }}"

# Binary download URLs
kubeadm_download_url: "{{ files_repo }}/kubernetes/v{{ kube_version }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/v{{ kube_version }}/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/v{{ kube_version }}/kubelet"
etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-v{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
cni_download_url: "{{ files_repo }}/kubernetes/cni/cni-plugins-linux-{{ image_arch }}-v{{ cni_version }}.tgz"
crictl_download_url: "{{ files_repo }}/kubernetes/cri-tools/crictl-v{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
runc_download_url: "{{ files_repo }}/runc/v{{ runc_version }}/runc.{{ image_arch }}"
nerdctl_download_url: "{{ files_repo }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
containerd_download_url: "{{ files_repo }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
helm_download_url: "{{ files_repo }}/helm-v{{ helm_version }}-linux-{{ image_arch }}.tar.gz"
calicoctl_download_url: "{{ files_repo }}/kubernetes/calico/v{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
calico_crds_download_url: "{{ files_repo }}/kubernetes/calico/v{{ calico_version }}.tar.gz"
ciliumcli_download_url: "{{ files_repo }}/cilium-cli/v{{ cilium_cli_version }}/cilium-linux-{{ image_arch }}.tar.gz"
```

## Containerd Registry Mirrors for Image Pull

Configure mirrors so pods referencing docker.io, quay.io, etc. are redirected to the private registry:

```yaml
containerd_registries_mirrors:
  - prefix: "{{ registry_host }}"
    mirrors:
      - host: "http://{{ registry_host }}"
        capabilities: ["pull", "resolve"]
        skip_verify: true
  - prefix: "docker.io"
    mirrors:
      - host: "http://ADMIN_IP:35000"
        capabilities: ["pull", "resolve"]
        skip_verify: false
  - prefix: "registry-1.docker.io"
    mirrors:
      - host: "http://ADMIN_IP:35000"
        capabilities: ["pull", "resolve"]
        skip_verify: false
  - prefix: "quay.io"
    mirrors:
      - host: "http://ADMIN_IP:35000"
        capabilities: ["pull", "resolve"]
        skip_verify: false
```

Apply mirror configuration without full redeploy:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --tags containerd
```

Resulting configuration on each node:

```
/etc/containerd/certs.d/
├── 192.168.10.10:35000/hosts.toml
├── docker.io/hosts.toml
├── quay.io/hosts.toml
└── registry-1.docker.io/hosts.toml
```

## Cluster Deployment

```bash
cd kubespray-2.30.0

# Install dependencies from offline pypi mirror
pip install -U pip && pip install -r requirements.txt

# Place offline.yml
cp offline.yml inventory/mycluster/group_vars/all/

# Configure inventory.ini and group_vars as needed

# Run offline-repo playbook (configures yum/apt repos on all nodes)
ansible-playbook -i inventory/mycluster/inventory.ini offline-repo/playbook/offline-repo.yml

# Deploy the cluster
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml
```

## Pushing Additional Images Post-Deploy

To add images after the cluster is running (e.g., application images):

```bash
# On admin server with podman
podman pull nginx:alpine
podman tag nginx:alpine ADMIN_IP:35000/library/nginx:alpine

# Configure insecure registry in /etc/containers/registries.conf if needed
podman push ADMIN_IP:35000/library/nginx:alpine
```

## Changing kube_version

To deploy a different Kubernetes version than the default:

```bash
# Modify generate_list.sh to target a specific version
sed -i 's|offline/generate_list.sh|offline/generate_list.sh -e kube_version=1.33.7|g' download-kubespray-files.sh

# Re-run to download version-specific binaries and images
./download-kubespray-files.sh
```

## Troubleshooting

### etcd download 404 on ARM64

```
etcd-v3.5.26-linux-amd64.tar.gz 404 Not Found
```

**Cause:** `amd64` is hardcoded instead of using architecture variable.

**Fix:** In `offline.yml`, ensure etcd URL uses `{{ image_arch }}`:

```yaml
etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-v{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
```

### ImagePullBackOff for docker.io Images

```
Failed to pull image "docker.io/library/nginx:1.25": rpc error: ...
```

**Fix:** Either push the image to the private registry, or configure `containerd_registries_mirrors` to redirect docker.io to the private registry. Then apply:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --tags containerd
```

### "image might be filtered out" During load-push

```
FATA[0000] image might be filtered out
```

**Cause:** Platform mismatch when loading multi-arch images.

**Fix:** Add `--all-platforms` to the nerdctl load command:

```bash
./load-push-all-images.sh --all-platforms
```

### loadFlannelSubnetEnv Failed

```
failed to load subnet env: open /run/flannel/subnet.env: no such file or directory
```

**Fix:** Flannel requires `/run/flannel/subnet.env`. Check that the CNI plugin is correctly installed and flannel DaemonSet is running.

### Nodes Cannot Resolve Hostnames

**Fix:** Configure DNS within the air-gapped network, or add entries to `/etc/hosts` on all nodes for the admin server and other cluster nodes.

### Apply Mirror Config Changes Only

Use the `--tags containerd` flag to apply containerd mirror configuration changes without running the full cluster deployment:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --tags containerd
```

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Missing images discovered late | DaemonSets/Deployments fail after deploy. Verify all images in `images.list` are pushed to registry before cluster deployment. |
| Architecture mismatch (amd64 vs arm64) | Use `{{ image_arch }}` in all download URLs. Use `--all-platforms` with nerdctl load on ARM64. |
| Registry not reachable from nodes | Ensure firewall allows port 35000 from all nodes to admin server. Test with `curl http://ADMIN_IP:35000/v2/`. |
| Forgot to run offline-repo playbook | Nodes cannot install OS packages. Run `offline-repo.yml` before `cluster.yml`. |
| Version mismatch between config.sh and kubespray | Keep `config.sh` versions aligned with the kubespray version specified in `KUBESPRAY_VERSION`. |
| DNS not configured in air-gap | Nodes fail to resolve admin server hostname. Use IP addresses or configure `/etc/hosts`. |
| Self-signed cert errors with registry | Use `skip_verify: true` in containerd mirrors, or distribute CA certs to all nodes. |
| Nginx file server not started | Binary downloads fail with connection refused. Run `./start-nginx.sh` before deployment. |
