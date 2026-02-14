---
name: kubespray-helm-airgap
description: Use when managing Helm charts in air-gapped environments - packaging charts as tgz, using OCI registries for Helm storage, setting up ChartMuseum, or deploying Helm releases from internal sources without internet access.
---

# Helm Chart Management in Air-Gapped Environments

## Overview

In air-gapped environments, Helm charts cannot be fetched from public repositories. Two approaches exist for managing Helm charts internally:

1. **Traditional Helm repo servers (ChartMuseum)** -- a dedicated HTTP server that hosts an `index.yaml` and packaged chart archives.
2. **OCI-compatible registries (modern)** -- reuses your existing container registry infrastructure (Harbor, Docker Registry, Zot) to store Helm charts as OCI artifacts.

Core principle: package Helm charts and their referenced container images together, then stage both in internal registries before deployment.

## When to Use

- Deploying Helm charts in air-gapped Kubernetes clusters
- Setting up internal Helm chart repositories
- Using an OCI registry for Helm chart storage
- Packaging and transferring Helm charts to isolated networks
- Setting up ChartMuseum in an air-gapped environment

## Helm Repo vs OCI Comparison

| Aspect | Helm Repo (Traditional) | OCI Registry (Modern) |
|--------|------------------------|----------------------|
| Storage | Dedicated Helm repo server | OCI-compatible container registry |
| Deploy command | `helm repo add` + `helm install` | `helm install oci://...` |
| Auth | Repo-specific auth | Docker registry auth reuse |
| Security | Separate security policies | Reuse existing registry policies |
| Pros | Familiar, widely compatible | No extra infra, CI/CD friendly, standardized |
| Cons | Extra server to maintain | Requires Helm 3.8+ |
| Example URL | `http://192.168.10.10:8080` | `oci://192.168.10.10:35000/helm-charts` |

## Creating and Packaging Helm Charts

Build a chart from scratch and package it for transfer:

```bash
mkdir nginx-chart && cd nginx-chart
mkdir templates

# Chart.yaml
cat > Chart.yaml <<EOF
apiVersion: v2
name: nginx-chart
description: A Helm chart for deploying Nginx
type: application
version: 1.0.0
appVersion: "1.28.0-alpine"
EOF

# values.yaml
cat > values.yaml <<EOF
image:
  repository: nginx
  tag: 1.28.0-alpine
replicaCount: 1
EOF

# Create templates: deployment.yaml, service.yaml, configmap.yaml, etc.

# Validate the rendered templates
helm template my-release . -f values.yaml

# Package the chart into a tgz archive
helm package .
# Output: nginx-chart-1.0.0.tgz
```

## OCI Registry for Helm Charts

Push charts to an existing OCI-compatible container registry (the same one used for Kubernetes images):

```bash
# Push chart to OCI registry
helm push nginx-chart-1.0.0.tgz oci://192.168.10.10:35000/helm-charts
# Pushed: 192.168.10.10:35000/helm-charts/nginx-chart:1.0.0

# Verify the chart is stored
curl -s 192.168.10.10:35000/v2/_catalog | jq | grep helm
curl -s 192.168.10.10:35000/v2/helm-charts/nginx-chart/tags/list | jq

# Install from OCI registry
helm install my-nginx oci://192.168.10.10:35000/helm-charts/nginx-chart --version 1.0.0

# Inspect chart metadata and values
helm show chart oci://192.168.10.10:35000/helm-charts/nginx-chart --version 1.0.0
helm show values oci://192.168.10.10:35000/helm-charts/nginx-chart --version 1.0.0
```

## ChartMuseum Setup

Deploy a ChartMuseum instance as a traditional Helm repository server:

```bash
# Create storage directory
mkdir -p /data/chartmuseum/charts
chmod 777 /data/chartmuseum/charts

# Run ChartMuseum container
podman run -d \
  --name chartmuseum \
  -p 8080:8080 \
  -v /data/chartmuseum/charts:/charts \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -e DEBUG=true \
  ghcr.io/helm/chartmuseum:v0.16.4

# Verify health
curl -s http://192.168.10.10:8080/health | jq
# {"healthy": true}

# Register as a Helm repo
helm repo add internal http://192.168.10.10:8080
helm repo update

# Install the helm-push plugin (required for cm-push)
helm plugin install https://github.com/chartmuseum/helm-push.git

# Push chart to ChartMuseum
helm cm-push nginx-chart-1.0.0.tgz internal
# Done.

# Install from ChartMuseum
helm repo update
helm install my-nginx internal/nginx-chart
```

## Using External Charts Offline

Pull charts from public registries on an internet-connected machine, then transfer them to the air-gapped environment:

```bash
# On internet-connected machine:
# Pull chart from public OCI registry
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 22.4.7
# Output: nginx-22.4.7.tgz

# Inspect the chart contents
tar -tzf nginx-22.4.7.tgz
zcat nginx-22.4.7.tgz | tar -xOf - nginx/Chart.yaml
zcat nginx-22.4.7.tgz | tar -xOf - nginx/values.yaml

# Transfer the tgz file to the air-gap admin server (USB, SCP over bastion, etc.)

# In air-gap: install directly from tgz
helm install my-nginx ./nginx-22.4.7.tgz --set service.type=NodePort

# Or push to internal OCI registry or ChartMuseum first
helm push nginx-22.4.7.tgz oci://192.168.10.10:35000/helm-charts
```

## Container Image Staging for Charts

Charts reference container images. You must stage those images in the internal registry as well, or pods will fail to pull:

```bash
# Check what images a chart needs
zcat nginx-22.4.7.tgz | tar -xOf - nginx/Chart.yaml | grep image:

# Pull, tag, and push images to internal registry
podman pull docker.io/bitnami/nginx:latest
podman tag bitnami/nginx:latest 192.168.10.10:35000/bitnami/nginx:latest

# Configure insecure registry if needed (for HTTP registries)
cat <<EOF >> /etc/containers/registries.conf
[[registry]]
location = "192.168.10.10:35000"
insecure = true
EOF

podman push 192.168.10.10:35000/bitnami/nginx:latest
```

When installing the chart, override image references to point to the internal registry:

```bash
helm install my-nginx ./nginx-22.4.7.tgz \
  --set image.repository=192.168.10.10:35000/bitnami/nginx \
  --set image.tag=latest
```

## Quick Reference

| Task | Command |
|------|---------|
| Package chart | `helm package .` |
| Push to OCI | `helm push chart.tgz oci://REGISTRY/path` |
| Install from OCI | `helm install NAME oci://REGISTRY/path/chart --version VER` |
| Push to ChartMuseum | `helm cm-push chart.tgz REPO_NAME` |
| Install from ChartMuseum | `helm install NAME REPO/chart` |
| Pull public chart | `helm pull oci://registry-1.docker.io/bitnamicharts/NAME` |
| Install from tgz | `helm install NAME ./chart.tgz` |
| Show chart info | `helm show chart oci://REGISTRY/path/chart` |

## Common Mistakes

- **Forgetting to push container images referenced by the chart.** The chart deploys successfully but pods fail with `ImagePullBackOff` because the images are not in the internal registry.
- **Not configuring insecure registry in `/etc/containers/registries.conf`.** Podman push to an HTTP registry will fail without this configuration.
- **Missing the helm-push plugin for ChartMuseum.** The `helm cm-push` command requires the plugin installed via `helm plugin install https://github.com/chartmuseum/helm-push.git`.
- **Using `helm repo` commands with OCI registries.** OCI registries do not use `helm repo add`. Use `helm push` and `helm install oci://` directly.
- **Chart values referencing public image repositories.** Override image references at install time with `--set image.repository=INTERNAL_REGISTRY/image` to point to the internal registry.
