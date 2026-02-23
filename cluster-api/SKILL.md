---
name: cluster-api
description: Use when managing Kubernetes clusters as Kubernetes resources with Cluster API (CAPI), provisioning workload clusters from a management cluster, performing declarative upgrades, or working with ClusterClass blueprints. Use when seeing "failed to connect to management cluster" or clusterctl errors.
---

# Cluster API (CAPI)

## Overview

Cluster API (CAPI) manages Kubernetes clusters as Kubernetes resources. Instead of SSH-ing into nodes and running imperative commands, you define clusters declaratively using Custom Resources, and CAPI controllers reconcile the desired state into reality.

**Core principle:** CAPI turns cluster lifecycle management into a Kubernetes-native workflow -- define a Cluster CR, apply it, and controllers handle provisioning, scaling, and upgrades automatically.

## When to Use

- Provisioning Kubernetes clusters declaratively from a management cluster
- Managing fleet of clusters at scale (tens to hundreds)
- Performing rolling upgrades by patching a version field
- Using ClusterClass blueprints for standardized cluster templates
- Setting up dev/test environments with the Docker infrastructure provider

**Not for:** Single cluster installation with security hardening (use rke2-deployment or kubespray-deployment), air-gapped environments without CAPI controller images, bare-metal provisioning without an infrastructure provider

## Key Concepts

| Concept | Description |
|---------|-------------|
| Management Cluster | Runs CAPI controllers, stores cluster definitions -- the "brain" |
| Workload Cluster | Provisioned BY CAPI, runs actual applications |
| Cluster CR | Defines desired cluster state (version, topology, networking) |
| Machine | Represents a single node (VM or container) |
| MachineDeployment | Manages a set of Machines (like Deployment manages Pods) |
| MachineSet | Intermediate resource between MachineDeployment and Machine |
| ClusterClass | Reusable blueprint for creating clusters with consistent topology |

## CAPI vs Kubespray/RKE2

| Aspect | Kubespray / RKE2 | Cluster API |
|--------|-------------------|-------------|
| Model | Imperative (Ansible playbooks / CLI) | Declarative (Kubernetes CRs) |
| Scope | Single cluster installation | Fleet management of many clusters |
| Upgrades | Manual per-node playbook runs | Patch version field, controllers handle it |
| Scale | One cluster at a time | Manage hundreds of clusters from one place |
| Security focus | RKE2: FIPS, SELinux, CIS hardened | Depends on infrastructure provider |
| Air-gap support | Strong (Kubespray offline, RKE2 embedded) | Requires CAPI controller images available |

## Provider Architecture

CAPI uses a modular provider architecture. Four provider types work together to provision and manage clusters.

```
  Management Cluster
  +------------------------------------------------------------------+
  |                                                                  |
  |  +-------------------+    +--------------------------------+     |
  |  | Core Provider     |    | Bootstrap Provider             |     |
  |  | (capi-system)     |    | (capi-kubeadm-bootstrap-system)|    |
  |  |                   |    |                                |     |
  |  | Manages:          |    | Generates:                     |     |
  |  |  - Cluster        |    |  - cloud-init / user-data     |     |
  |  |  - Machine        |    |  - kubeadm init config        |     |
  |  |  - MachineSet     |    |  - kubeadm join config        |     |
  |  |  - MachineDeploymt|    +--------------------------------+     |
  |  +-------------------+                                           |
  |                                                                  |
  |  +---------------------------+  +----------------------------+   |
  |  | Control Plane Provider    |  | Infrastructure Provider    |   |
  |  | (capi-kubeadm-control-    |  | (e.g., capd-system for     |   |
  |  |  plane-system)            |  |  Docker, capv for vSphere, |   |
  |  |                           |  |  capa for AWS, etc.)       |   |
  |  | Manages:                  |  |                            |   |
  |  |  - KubeadmControlPlane   |  | Creates:                   |   |
  |  |  - CP scaling/upgrades   |  |  - Actual VMs / containers |   |
  |  |  - etcd membership       |  |  - Load balancers          |   |
  |  +---------------------------+  |  - Networks / disks        |   |
  |                                 +----------------------------+   |
  +------------------------------------------------------------------+
```

### Provider Types

| Provider | Namespace | Responsibility |
|----------|-----------|---------------|
| Core | `capi-system` | Manages Cluster, Machine, MachineSet, MachineDeployment CRDs |
| Bootstrap | `capi-kubeadm-bootstrap-system` | Generates cloud-init/user-data for kubeadm init/join |
| Control Plane | `capi-kubeadm-control-plane-system` | Manages KubeadmControlPlane, handles CP scaling/upgrades/etcd |
| Infrastructure | Provider-specific (e.g., `capd-system`) | Creates actual VMs/containers, LBs, networks |

### Infrastructure Provider Examples

| Provider | Prefix | Creates |
|----------|--------|---------|
| Docker (CAPD) | `capd-system` | Docker containers simulating nodes (dev/test) |
| vSphere (CAPV) | `capv-system` | vSphere VMs |
| AWS (CAPA) | `capa-system` | EC2 instances, ELBs, VPCs |
| Azure (CAPZ) | `capz-system` | Azure VMs, LBs, VNets |
| Metal3 (CAPM3) | `capm3-system` | Bare-metal servers via Ironic |

## Management Cluster vs Workload Cluster

```
  +-------------------------------------------+
  |         Management Cluster (KinD)         |
  |                                           |
  |  CAPI Controllers:                        |
  |    - Core Provider                        |
  |    - Bootstrap Provider                   |
  |    - Control Plane Provider               |
  |    - Infrastructure Provider              |
  |    - cert-manager                         |
  |                                           |
  |  Stores CRs:                              |
  |    - Cluster/capi-quickstart              |
  |    - KubeadmControlPlane/...              |
  |    - MachineDeployment/...                |
  |    - Machine/... (one per node)           |
  +-------------------------------------------+
       |              |              |
       | reconcile    | reconcile    | reconcile
       v              v              v
  +-------------------------------------------+
  |        Workload Cluster (capi-quickstart) |
  |                                           |
  |  +----------+  +---------+  +---------+   |
  |  |  CP-1    |  |  CP-2   |  |  CP-3   |   |
  |  | (node)   |  | (node)  |  | (node)  |   |
  |  +----------+  +---------+  +---------+   |
  |                                           |
  |  +----------+  +---------+  +---------+   |
  |  | Worker-1 |  | Worker-2|  | Worker-3|   |
  |  | (node)   |  | (node)  |  | (node)  |   |
  |  +----------+  +---------+  +---------+   |
  |                                           |
  |  HAProxy LB (for API server access)       |
  |                                           |
  |  Runs actual application workloads        |
  +-------------------------------------------+
```

**Key distinction:**
- Management cluster is the "brain" -- it runs CAPI controllers and stores cluster definitions
- Workload clusters are the "muscle" -- they run actual application workloads
- The management cluster does NOT run application pods
- One management cluster can manage many workload clusters

## Hands-On Setup (Docker Provider)

The Docker provider (CAPD) simulates nodes as Docker containers. Ideal for learning, development, and CI/CD testing.

### Step 1: Create Management Cluster with KinD

```bash
kind create cluster --name myk8s --image kindest/node:v1.35.0 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
EOF
```

**Why these settings:**
- `extraMounts` for Docker socket -- CAPD needs access to the host Docker daemon to create containers that simulate workload cluster nodes
- `extraPortMappings` -- maps NodePort ranges so you can access workload cluster services from your host machine

### Step 2: Install clusterctl

```bash
# macOS
brew install clusterctl

# Linux (amd64)
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/latest/download/clusterctl-linux-amd64 -o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/
```

Verify installation:
```bash
clusterctl version
```

### Step 3: Initialize Management Cluster

```bash
# Enable ClusterClass/Topology support
export CLUSTER_TOPOLOGY=true

# Initialize with Docker infrastructure provider
clusterctl init --infrastructure docker
```

**What this installs:**
1. **cert-manager** -- manages TLS certificates for CAPI webhooks
2. **Core provider** (`capi-system`) -- Cluster, Machine, MachineSet, MachineDeployment CRDs
3. **Bootstrap provider** (`capi-kubeadm-bootstrap-system`) -- generates kubeadm configs
4. **Control Plane provider** (`capi-kubeadm-control-plane-system`) -- manages KubeadmControlPlane
5. **Infrastructure provider** (`capd-system`) -- Docker-specific controllers

### Step 4: Verify Initialization

```bash
# Check all providers are installed and ready
kubectl get providers.clusterctl.cluster.x-k8s.io -A
```

Expected output:
```
NAMESPACE                           NAME                    TYPE                     VERSION   WATCH NAMESPACE
capd-system                         infrastructure-docker   InfrastructureProvider   v1.x.x
capi-kubeadm-bootstrap-system       bootstrap-kubeadm      BootstrapProvider        v1.x.x
capi-kubeadm-control-plane-system   control-plane-kubeadm  ControlPlaneProvider     v1.x.x
capi-system                         cluster-api             CoreProvider             v1.x.x
```

```bash
# Check all controller pods are running
kubectl get pods -A | grep -E 'capi|capd|cert-manager'
```

All pods should be in `Running` state with `1/1` or `2/2` ready.

## Provisioning a Workload Cluster

### Generate Cluster Manifest

```bash
# Set networking variables
export SERVICE_CIDR='["10.20.0.0/16"]'
export POD_CIDR='["10.10.0.0/16"]'
export SERVICE_DOMAIN="myk8s-1.local"
export POD_SECURITY_STANDARD_ENABLED="false"

# Generate cluster manifest
clusterctl generate cluster capi-quickstart --flavor development \
  --kubernetes-version v1.34.3 \
  --control-plane-machine-count=3 \
  --worker-machine-count=3 \
  > capi-quickstart.yaml
```

**What `--flavor development` does:** Selects a pre-built template designed for local Docker-based development. The template includes ClusterClass definitions, DockerCluster/DockerMachine templates, and kubeadm configurations.

### Apply Cluster Manifest

```bash
kubectl apply -f capi-quickstart.yaml
```

### Monitor Provisioning

```bash
# Watch cluster provisioning status
clusterctl describe cluster capi-quickstart

# Watch in real-time
clusterctl describe cluster capi-quickstart --show-conditions all
```

**What happens during provisioning (Docker provider):**
1. CAPI creates Docker containers using `kindest/node` image to simulate cluster nodes
2. An HAProxy container is created as the load balancer for the workload cluster API server
3. Bootstrap provider generates kubeadm init/join configurations
4. Control plane provider runs kubeadm init on the first CP node, then joins remaining CP nodes
5. Worker machines are bootstrapped with kubeadm join

```
  Provisioning Flow (Docker Provider)
  +--------------------------------------------------------------------+
  |                                                                    |
  |  kubectl apply -f capi-quickstart.yaml                             |
  |       |                                                            |
  |       v                                                            |
  |  Core Provider: creates Cluster object                             |
  |       |                                                            |
  |       v                                                            |
  |  Infrastructure Provider (CAPD):                                   |
  |    - Creates DockerCluster (networking setup)                      |
  |    - Creates HAProxy container (LB for API server)                 |
  |       |                                                            |
  |       v                                                            |
  |  Control Plane Provider:                                           |
  |    - Creates first Machine (Docker container)                      |
  |    - Bootstrap Provider generates kubeadm init config              |
  |    - Runs kubeadm init inside container                            |
  |    - Repeats for CP nodes 2 and 3 (kubeadm join --control-plane)  |
  |       |                                                            |
  |       v                                                            |
  |  MachineDeployment controller:                                     |
  |    - Creates MachineSet -> Machines for workers                    |
  |    - Bootstrap Provider generates kubeadm join config              |
  |    - Runs kubeadm join inside each worker container                |
  |       |                                                            |
  |       v                                                            |
  |  Cluster status: Provisioned                                       |
  |  (Nodes are NotReady until CNI is installed)                       |
  +--------------------------------------------------------------------+
```

### Access the Workload Cluster

```bash
# Get kubeconfig for the workload cluster
clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
```

**Fix kubeconfig for local access (Docker provider only):**

The generated kubeconfig contains the internal Docker network IP for the API server. To access from your host, you must replace it with `127.0.0.1` and the mapped port.

```bash
# Find the HAProxy container's mapped port
docker ps | grep capi-quickstart-lb

# Example output:
#   ... 0.0.0.0:43210->6443/tcp ... capi-quickstart-lb
# The mapped port is 43210

# Update kubeconfig to use localhost and mapped port
sed -i 's|server: https://.*:6443|server: https://127.0.0.1:43210|' capi-quickstart.kubeconfig

# macOS:
sed -i '' 's|server: https://.*:6443|server: https://127.0.0.1:43210|' capi-quickstart.kubeconfig
```

### Install CNI (Required)

Nodes remain in `NotReady` state until a CNI plugin is installed. CAPI does not install CNI automatically -- this is by design, giving you flexibility to choose your CNI.

```bash
# Install Calico CNI
kubectl --kubeconfig=capi-quickstart.kubeconfig \
  apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

**Verify nodes transition to Ready:**
```bash
# Watch nodes go from NotReady -> Ready
kubectl --kubeconfig=capi-quickstart.kubeconfig get nodes -w
```

All nodes should show `Ready` status within 1-2 minutes after CNI installation.

### Full Verification

```bash
# All nodes Ready
kubectl --kubeconfig=capi-quickstart.kubeconfig get nodes -o wide

# System pods Running
kubectl --kubeconfig=capi-quickstart.kubeconfig get pods -A

# From management cluster: cluster status
clusterctl describe cluster capi-quickstart

# Check Machine objects
kubectl get machines -o wide

# Check MachineDeployments
kubectl get machinedeployments -o wide
```

## ClusterClass (Topology)

ClusterClass is a reusable blueprint for creating clusters. Define the cluster shape once (templates for infrastructure, control plane, workers), then reference it when creating clusters.

### Enable ClusterClass

```bash
# Must be set before clusterctl init
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker
```

### ClusterClass Resource Hierarchy

```
  ClusterClass (blueprint)
  +------------------------------------------------------------+
  |  name: my-cluster-class                                    |
  |                                                            |
  |  Infrastructure:                                           |
  |    ref: DockerClusterTemplate/my-cluster-class             |
  |                                                            |
  |  Control Plane:                                            |
  |    ref: KubeadmControlPlaneTemplate/my-cluster-class       |
  |    Infrastructure:                                         |
  |      ref: DockerMachineTemplate/my-cluster-class-cp        |
  |                                                            |
  |  Workers:                                                  |
  |    MachineDeployments:                                     |
  |    - class: default-worker                                 |
  |      template:                                             |
  |        bootstrap:                                          |
  |          ref: KubeadmConfigTemplate/my-cluster-class       |
  |        infrastructure:                                     |
  |          ref: DockerMachineTemplate/my-cluster-class-wkr   |
  +------------------------------------------------------------+

  Cluster (instance using ClusterClass)
  +------------------------------------------------------------+
  |  name: capi-quickstart                                     |
  |  spec:                                                     |
  |    topology:                                               |
  |      class: my-cluster-class    <-- references ClusterClass|
  |      version: v1.34.3                                      |
  |      controlPlane:                                         |
  |        replicas: 3                                         |
  |      workers:                                              |
  |        machineDeployments:                                 |
  |        - class: default-worker                             |
  |          name: md-0                                        |
  |          replicas: 3                                       |
  +------------------------------------------------------------+
```

### Resource Types in ClusterClass

| Resource | Purpose |
|----------|---------|
| `ClusterClass` | Top-level blueprint tying all templates together |
| `DockerClusterTemplate` | Template for infrastructure-level cluster config (network, LB) |
| `KubeadmControlPlaneTemplate` | Template for control plane behavior (kubeadm config, rollout) |
| `DockerMachineTemplate` | Template for individual machine specs (CPU, memory, image) |
| `KubeadmConfigTemplate` | Template for worker node bootstrap configuration |
| `Cluster` | Instance that references a ClusterClass via `spec.topology.class` |

### Benefits of ClusterClass

- **Consistency:** All clusters from the same class share identical infrastructure and bootstrap configuration
- **Simplicity:** Creating a new cluster requires only a small Cluster CR referencing the class
- **Upgrades:** Update the ClusterClass and all clusters using it can be rolled forward
- **Governance:** Platform teams define ClusterClass, application teams consume them

## Declarative Cluster Upgrade

One of CAPI's most powerful features: upgrade an entire cluster by patching a single field.

### Upgrade from v1.34 to v1.35

```bash
kubectl patch cluster capi-quickstart --type merge \
  -p '{"spec":{"topology":{"version":"v1.35.0"}}}'
```

**That is the entire upgrade command.** One line. No SSH, no manual kubeadm commands, no node-by-node playbook runs.

### What Happens During Upgrade

```
  Declarative Upgrade Flow
  +--------------------------------------------------------------------+
  |                                                                    |
  |  kubectl patch cluster ... version: "v1.35.0"                      |
  |       |                                                            |
  |       v                                                            |
  |  Control Plane Provider detects version drift                      |
  |       |                                                            |
  |       v                                                            |
  |  Phase 1: Control Plane Rolling Upgrade                            |
  |    For each CP node (one at a time):                               |
  |      1. Create new Machine with v1.35.0 image                      |
  |      2. Wait for new node to join and become Ready                 |
  |      3. Cordon old node                                            |
  |      4. Drain old node (evict workloads)                           |
  |      5. Remove old Machine (delete container/VM)                   |
  |      6. Update etcd membership                                     |
  |    Repeat until all CP nodes are on v1.35.0                        |
  |       |                                                            |
  |       v                                                            |
  |  Phase 2: Worker Rolling Upgrade                                   |
  |    MachineDeployment controller:                                   |
  |      1. Creates new MachineSet with v1.35.0                        |
  |      2. Scales up new MachineSet (new workers join)                |
  |      3. Cordons old workers                                        |
  |      4. Drains old workers                                         |
  |      5. Scales down old MachineSet (removes old workers)           |
  |    Proceeds according to rollout strategy (default: rolling)       |
  |       |                                                            |
  |       v                                                            |
  |  Upgrade complete: all nodes running v1.35.0                       |
  +--------------------------------------------------------------------+
```

### Monitor Upgrade Progress

```bash
# Watch the upgrade in real-time
clusterctl describe cluster capi-quickstart --show-conditions all

# Watch Machine replacements
kubectl get machines -w

# Check rollout status of MachineDeployments
kubectl get machinedeployments -o wide

# Verify versions on workload cluster
kubectl --kubeconfig=capi-quickstart.kubeconfig get nodes -o wide
```

### Upgrade Behavior Details

| Aspect | Behavior |
|--------|----------|
| CP upgrade order | One node at a time (serial), new created before old removed |
| Worker upgrade order | Rolling update via MachineDeployment (configurable surge/unavailable) |
| etcd handling | Automatic -- membership updated as CP nodes are replaced |
| Workload disruption | Pods drained gracefully before node removal |
| Rollback | Not automatic -- patch version back to previous if needed |
| Failure handling | Upgrade pauses if a new node fails to become Ready |

## Cleanup

### Delete Workload Cluster

```bash
# Delete the workload cluster (from management cluster context)
kubectl delete cluster capi-quickstart
```

This single command removes ALL resources associated with the workload cluster:
- All Machine objects (Docker containers or VMs)
- The load balancer container
- MachineDeployments, MachineSets
- Bootstrap configs, kubeadm configs
- Infrastructure resources (DockerCluster, DockerMachines)

**CAPI uses owner references** -- deleting the Cluster object cascades to all child resources automatically.

### Delete Management Cluster

```bash
# Remove the KinD management cluster
kind delete cluster --name myk8s
```

## Custom Resource Quick Reference

| Resource | API Group | Managed By | Purpose |
|----------|-----------|------------|---------|
| `Cluster` | `cluster.x-k8s.io` | Core Provider | Top-level cluster definition |
| `Machine` | `cluster.x-k8s.io` | Core Provider | Single node representation |
| `MachineSet` | `cluster.x-k8s.io` | Core Provider | Manages a group of identical Machines |
| `MachineDeployment` | `cluster.x-k8s.io` | Core Provider | Declarative updates for MachineSets |
| `KubeadmControlPlane` | `controlplane.cluster.x-k8s.io` | CP Provider | Control plane scaling and upgrades |
| `KubeadmConfig` | `bootstrap.cluster.x-k8s.io` | Bootstrap Provider | kubeadm bootstrap configuration |
| `KubeadmConfigTemplate` | `bootstrap.cluster.x-k8s.io` | Bootstrap Provider | Template for worker bootstrap configs |
| `DockerCluster` | `infrastructure.cluster.x-k8s.io` | Infra Provider | Docker-specific cluster infrastructure |
| `DockerMachine` | `infrastructure.cluster.x-k8s.io` | Infra Provider | Docker-specific machine (container) |
| `DockerMachineTemplate` | `infrastructure.cluster.x-k8s.io` | Infra Provider | Template for DockerMachine specs |
| `ClusterClass` | `cluster.x-k8s.io` | Core Provider | Reusable cluster blueprint |

## Useful Commands

| Command | Purpose |
|---------|---------|
| `clusterctl init --infrastructure docker` | Initialize management cluster with Docker provider |
| `clusterctl generate cluster <name> ...` | Generate cluster manifest from templates |
| `clusterctl describe cluster <name>` | Show cluster topology and status |
| `clusterctl get kubeconfig <name>` | Retrieve workload cluster kubeconfig |
| `clusterctl move --to-kubeconfig <path>` | Move CAPI objects to a different management cluster |
| `kubectl get clusters` | List all CAPI-managed clusters |
| `kubectl get machines -o wide` | List all machines with status and versions |
| `kubectl get machinedeployments` | List all MachineDeployments |
| `kubectl get kubeadmcontrolplane` | List control plane objects with replica counts |

## Common Errors (Searchable)

```
error: failed to connect to the management cluster
```
**Cause:** kubectl context not pointing to the management cluster. **Fix:** Run `kubectl config use-context kind-myk8s` to switch to the management cluster context.

```
Error: action failed after X attempts: failed to get provider components for the "docker" provider
```
**Cause:** clusterctl cannot download provider components (network issue or wrong version). **Fix:** Check internet connectivity. For air-gapped setups, use `clusterctl init` with local provider repository configuration in `~/.cluster-api/clusterctl.yaml`.

```
Cluster "capi-quickstart" is not ready: WaitForControlPlaneAvailable
```
**Cause:** Control plane machines are still provisioning or failing to bootstrap. **Fix:** Check Machine objects (`kubectl get machines`), inspect Machine conditions, check Docker container logs (`docker logs <container-name>`).

```
Unable to connect to the server: x509: certificate signed by unknown authority
```
**Cause:** Kubeconfig for workload cluster has incorrect CA or server address. **Fix:** Re-retrieve kubeconfig with `clusterctl get kubeconfig` and fix the server address for local Docker access.

```
nodes are not ready: node "capi-quickstart-xxxx" not Ready
```
**Cause:** CNI not installed on workload cluster. **Fix:** Install a CNI plugin (Calico, Cilium, Flannel) using the workload cluster's kubeconfig. Nodes remain NotReady until CNI is deployed.

```
Error from server: error when creating "capi-quickstart.yaml": admission webhook "validation.cluster.cluster.x-k8s.io" denied the request: Cluster.spec.topology.class "quick-start" not found
```
**Cause:** ClusterClass referenced in the manifest does not exist. **Fix:** Ensure `CLUSTER_TOPOLOGY=true` was set before `clusterctl init`. The `--flavor development` flag includes ClusterClass definitions in the generated manifest -- make sure to apply the full file.

```
docker: Error response from daemon: Conflict. The container name "/capi-quickstart-lb" is already in use
```
**Cause:** Leftover Docker containers from a previous CAPI cluster with the same name. **Fix:** Delete the old cluster first (`kubectl delete cluster capi-quickstart`), or remove orphaned containers manually (`docker rm -f capi-quickstart-lb`).

```
Error: timed out waiting for the condition on machines
```
**Cause:** Machine provisioning is stuck -- could be Docker resource limits, image pull failures, or kubeadm init/join failures. **Fix:** Check Docker daemon resources (`docker system df`), inspect the Machine's bootstrap data, and check container logs for kubeadm errors.

```
failed to create kind cluster: failed to init node: kubeadm init failed
```
**Cause:** KinD management cluster creation failed, often due to insufficient Docker resources or conflicting port mappings. **Fix:** Ensure Docker has sufficient memory (at least 4 GB for management + workload clusters), check port availability for extraPortMappings.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Forgetting `CLUSTER_TOPOLOGY=true` before `clusterctl init` | ClusterClass CRDs not installed; re-run `clusterctl init` with the env var set |
| Not mounting Docker socket in KinD config | CAPD cannot create workload cluster containers; recreate KinD with `extraMounts` for `/var/run/docker.sock` |
| Skipping CNI installation on workload cluster | All nodes stuck in `NotReady`, pods cannot be scheduled |
| Using generated kubeconfig directly (Docker provider) | Connection refused; replace API server address with `127.0.0.1:<mapped-port>` |
| Deleting Docker containers manually instead of `kubectl delete cluster` | CAPI state becomes inconsistent, orphaned resources in management cluster |
| Running `kubectl delete machine` directly | CAPI will reconcile and recreate the Machine; use MachineDeployment replicas or `kubectl delete cluster` |
| Trying to SSH into CAPD Docker containers | Containers are not VMs; use `docker exec -it <container-name> bash` instead |
| Not checking CAPI provider versions match | Version mismatch causes reconciliation failures; use `clusterctl upgrade plan` to check |
| Insufficient Docker resources for management + workload clusters | Containers OOM-killed; allocate at least 4 GB memory, 8 GB recommended for 3+3 clusters |
| Ignoring `clusterctl describe` output during provisioning | Missing early error signals; always monitor with `clusterctl describe cluster <name> --show-conditions all` |
