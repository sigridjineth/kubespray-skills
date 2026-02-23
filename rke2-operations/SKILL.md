---
name: rke2-operations
description: Use when managing RKE2 cluster certificates, performing manual or automated version upgrades, rotating TLS certificates, deploying the System Upgrade Controller, or troubleshooting RKE2 certificate and upgrade errors. Use when seeing "x509 certificate has expired" or "CertificateExpirationWarning" events or "Job has reached the specified backoff limit" errors.
---

# RKE2 Operations

## Overview

RKE2 is a FIPS-compliant Kubernetes distribution that manages its own TLS certificates and provides built-in upgrade mechanisms. Understanding certificate lifecycle and upgrade procedures is essential for maintaining cluster health and security.

**Core principle:** Always upgrade control plane (server) nodes before worker (agent) nodes. Never skip certificate inspection before rotation, and never skip pre-upgrade health checks before version upgrades.

## When to Use

- Inspecting or rotating RKE2 TLS certificates
- Upgrading RKE2 cluster versions (manual or automated)
- Deploying the System Upgrade Controller for automated rolling upgrades
- Troubleshooting certificate expiration warnings or TLS errors
- Planning maintenance windows for certificate or version operations

**Not for:** Initial RKE2 installation (use rke2-deployment), Kubespray-managed clusters (use kubespray-operations), Rancher UI-driven upgrades (use Rancher documentation)

## Certificate Management

### Certificate Validity

| Certificate Type | Default Validity | Notes |
|------------------|-----------------|-------|
| Client certificates | 365 days | API server, scheduler, controller-manager, kubelet, kube-proxy, etcd |
| Server certificates | 365 days | All serving certificates |
| CA certificates | 10 years | Root trust anchors, not rotated automatically |

All RKE2 components communicate over TLS. Every API call, etcd transaction, and kubelet heartbeat uses mutual TLS authentication.

### Auto-Renewal Behavior

RKE2 checks certificate expiration on every service start. If any certificate is within **120 days** of expiry, RKE2 automatically renews it during startup. Kubernetes also emits `CertificateExpirationWarning` events when certificates are less than 120 days from expiry.

**Implication:** If your cluster runs continuously without service restarts for longer than 245 days (365 minus 120), certificates will NOT be auto-renewed. Regular maintenance restarts or manual rotation are required.

### Inspecting Certificates

```bash
rke2 certificate check --output table
```

Output columns:

| Column | Description |
|--------|-------------|
| FILENAME | Path to the certificate file |
| SUBJECT | Certificate subject (CN and O fields) |
| USAGES | Key usage (client auth, server auth, or both) |
| EXPIRES | Expiration date and time |
| RESIDUAL TIME | Time remaining until expiry |
| STATUS | `ok` or `expiring` (within 120 days) |

### Server Node Certificates

A server (control plane) node holds certificates for all components:

| Component | Purpose |
|-----------|---------|
| kube-apiserver | API server serving and client certificates |
| kube-scheduler | Scheduler client certificate for API server auth |
| kube-controller-manager | Controller manager client certificate |
| kubelet | Kubelet client and serving certificates |
| kube-proxy | Proxy client certificate |
| etcd | etcd peer, server, and client certificates |
| rke2-supervisor | Supervisor API serving certificate |

### Agent Node Certificates

An agent (worker) node holds a smaller set:

| Component | Purpose |
|-----------|---------|
| kubelet | Kubelet client and serving certificates |
| kube-proxy | Proxy client certificate |
| rke2-controller | Agent controller client certificate |

## Manual Certificate Rotation

Use manual rotation when certificates are approaching expiry and you cannot rely on a service restart triggering auto-renewal, or when you need to rotate certificates immediately for security reasons.

### Step-by-Step Procedure

**Step 1: Stop the RKE2 server service**

```bash
systemctl stop rke2-server
```

**Step 2: Rotate all certificates**

```bash
rke2 certificate rotate
```

This command:
- Generates new certificates for all components
- Backs up old certificates to a timestamped directory (e.g., `/var/lib/rancher/rke2/server/tls-YYYY-MM-DDTHH-MM-SS/`)
- The backup allows rollback if anything goes wrong

**Step 3: Verify new certificate dates**

```bash
rke2 certificate check --output table
```

Confirm that EXPIRES column shows dates approximately 365 days from now and STATUS shows `ok` for all entries.

**Step 4: Start the RKE2 server service**

```bash
systemctl start rke2-server
```

**Step 5: Update your local kubeconfig**

The rotation generates a new admin client certificate embedded in `rke2.yaml`. Copy it to your working kubeconfig:

```bash
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

If accessing the cluster remotely, also update the `server:` field in the kubeconfig to the correct external address.

**Step 6: Verify cluster health**

```bash
# Nodes should be Ready
kubectl get nodes

# All system pods running
kubectl get pods -n kube-system

# API server responsive
kubectl get --raw='/readyz?verbose'
```

### Worker Node Behavior After Rotation

Worker (agent) nodes automatically reconnect to the server and receive new certificates. No manual action is required on agent nodes. The agent detects the trust chain change on its next heartbeat and re-enrolls.

### Multi-Server Rotation

For HA clusters with multiple server nodes, rotate certificates on each server node one at a time:

```bash
# On server-1
systemctl stop rke2-server
rke2 certificate rotate
rke2 certificate check --output table
systemctl start rke2-server
# Wait for server-1 to fully rejoin before proceeding

# On server-2
systemctl stop rke2-server
rke2 certificate rotate
rke2 certificate check --output table
systemctl start rke2-server
# Wait for server-2, then proceed to server-3, etc.
```

## Manual Version Upgrade

### Pre-Upgrade Monitoring

Before starting any upgrade, establish baseline monitoring in separate terminals:

```bash
# Terminal 1: Watch application availability
watch -n 2 'curl -s -o /dev/null -w "%{http_code}" http://<app-endpoint>'

# Terminal 2: Watch pod status
watch -n 2 'kubectl get pods -A -o wide'

# Terminal 3: Watch node status
watch -n 2 'kubectl get nodes -o wide'

# Terminal 4: Check etcd cluster health
ETCDCTL_API=3 etcdctl member list \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  --endpoints=https://127.0.0.1:2379
```

### Check Available Versions

```bash
curl -s https://update.rke2.io/v1-release/channels | jq '.data[] | {id, latest}'
```

This shows all release channels and their latest resolved versions.

### Version Skew Policy

Kubernetes 1.28+ supports a **3 minor version** skew between the control plane and worker nodes (earlier versions support 2). This means during an upgrade from v1.33 to v1.34, workers running v1.33 will continue to function normally while the control plane runs v1.34.

However, best practice is to upgrade workers promptly after the control plane to minimize the skew window.

### Upgrade Order

**Always upgrade server (control plane) nodes first, then agent (worker) nodes.**

```
server-1 (v1.33 -> v1.34)
server-2 (v1.33 -> v1.34)
server-3 (v1.33 -> v1.34)
  |
  v  (CP fully upgraded, then workers)
agent-1  (v1.33 -> v1.34)
agent-2  (v1.33 -> v1.34)
agent-3  (v1.33 -> v1.34)
```

### Server (Control Plane) Upgrade

**Step 1: Run the RKE2 installer with the target channel**

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.34 sh -
```

This upgrades the RPM packages in-place (`rke2-common`, `rke2-server`) without starting the service.

**Step 2: Restart the RKE2 server**

```bash
systemctl restart rke2-server
```

**Step 3: Verify the server is running the new version**

```bash
kubectl get nodes -o wide
# VERSION column should show the new Kubernetes version for this server node
```

**Step 4: Repeat for each additional server node**

Wait for each server node to fully rejoin and show `Ready` status before proceeding to the next server.

### Agent (Worker) Upgrade

After ALL server nodes are upgraded and healthy:

**Step 1: Run the RKE2 installer for the agent**

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=v1.34 sh -
```

This upgrades RPM packages (`rke2-common`, `rke2-agent`) in-place.

**Step 2: Restart the RKE2 agent**

```bash
systemctl restart rke2-agent
```

**Step 3: Verify the agent is running the new version**

```bash
kubectl get nodes -o wide
# VERSION column should show the new Kubernetes version for this agent node
```

**Step 4: Repeat for each additional agent node**

For production clusters, drain each agent before restarting and uncordon after:

```bash
# Drain the worker
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Upgrade and restart
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=v1.34 sh -
systemctl restart rke2-agent

# Uncordon after the node is Ready
kubectl uncordon <node-name>
```

### Post-Upgrade Verification

```bash
# All nodes on new version
kubectl get nodes -o wide

# All system pods running
kubectl get pods -n kube-system

# API server health
kubectl get --raw='/readyz?verbose'

# etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  --endpoints=https://127.0.0.1:2379

# Application still responding
curl -s -o /dev/null -w "%{http_code}" http://<app-endpoint>
```

## Automated Upgrade with System Upgrade Controller

The System Upgrade Controller (SUC) automates RKE2 version upgrades using Kubernetes-native Plan CRDs. It creates Jobs that run on each node to perform the actual upgrade.

### Install the System Upgrade Controller

**Step 1: Apply the CRD and controller manifests**

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml \
  -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

**Step 2: Verify the installation**

```bash
# Namespace created
kubectl get namespace system-upgrade

# Controller running
kubectl get deploy -n system-upgrade system-upgrade-controller

# CRD registered
kubectl get crd plans.upgrade.cattle.io
```

### What Gets Created

| Resource | Purpose |
|----------|---------|
| `system-upgrade` namespace | Isolates upgrade controller resources |
| `system-upgrade-controller` Deployment | Watches Plan CRDs and creates upgrade Jobs |
| `system-upgrade` ServiceAccount | Identity for the controller |
| `system-upgrade-controller` ClusterRoleBinding | Grants permissions to manage nodes and jobs |
| Drainer ClusterRole | Allows the controller to cordon and drain nodes |
| `plans.upgrade.cattle.io` CRD | Custom resource for defining upgrade plans |

### Upgrade Plans

Two Plan resources are needed: one for server nodes (control plane) and one for agent nodes (workers). The agent plan references the server plan in its `prepare` step, ensuring the control plane is fully upgraded before any worker upgrade begins.

#### Server Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: In
        values:
          - "true"
  serviceAccountName: system-upgrade
  tolerations:
    - key: CriticalAddonsOnly
      operator: Exists
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/etcd
      operator: Exists
      effect: NoExecute
  upgrade:
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/latest
```

Key fields:
- `concurrency: 1` -- Upgrade one server node at a time to maintain quorum
- `cordon: true` -- Mark node as unschedulable during upgrade
- `nodeSelector` -- Targets only nodes with the `node-role.kubernetes.io/control-plane: "true"` label
- `channel` -- The controller resolves the latest version from this URL
- `image: rancher/rke2-upgrade` -- Container image that performs the actual RKE2 binary upgrade

#### Agent Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 2
  cordon: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  prepare:
    image: rancher/rke2-upgrade
    args:
      - prepare
      - server-plan
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/latest
```

Key fields:
- `nodeSelector` with `DoesNotExist` -- Targets nodes WITHOUT the control-plane label (i.e., workers only)
- `prepare` step -- References `server-plan` by name; the agent plan waits until the server plan has completed on all server nodes before starting
- `concurrency: 2` -- Can upgrade two workers in parallel (adjust based on cluster capacity)

### Apply the Plans

```bash
kubectl apply -f server-plan.yaml
kubectl apply -f agent-plan.yaml
```

### Monitor Upgrade Progress

```bash
# Watch plan status
kubectl get plans -n system-upgrade -w

# Watch upgrade jobs
kubectl get jobs -n system-upgrade -w

# Check node versions as they upgrade
watch -n 5 'kubectl get nodes -o wide'
```

### How the Upgrade Works Internally

1. The controller reads the `channel` URL and resolves the latest version
2. For each node matching the plan's `nodeSelector`, the controller creates a Job
3. The upgrade pod runs with elevated privileges:
   - Mounts the host root filesystem (`/`) with read-write access
   - Uses host IPC, NET, and PID namespaces
   - Has `CAP_SYS_BOOT` capability (to reboot the node if needed)
4. The pod replaces RKE2 binaries on the host and restarts the RKE2 service
5. The node comes back with the new version
6. The controller marks the node as upgraded and proceeds to the next

### Cleanup of System Upgrade Controller

After the upgrade is complete and verified, remove the SUC resources:

```bash
# Delete the plans first
kubectl delete plan -n system-upgrade server-plan agent-plan

# Delete the controller and RBAC
kubectl delete -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml

# Delete the CRD (removes all Plan resources if any remain)
kubectl delete -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml
```

Verify cleanup:

```bash
# Namespace should be gone or empty
kubectl get all -n system-upgrade

# CRD should be removed
kubectl get crd plans.upgrade.cattle.io
# Expected: Error from server (NotFound)
```

## Quick Reference

### Certificate Commands

| Action | Command |
|--------|---------|
| Inspect all certificates | `rke2 certificate check --output table` |
| Rotate all certificates | `rke2 certificate rotate` (with service stopped) |
| Check certificate events | `kubectl get events --field-selector reason=CertificateExpirationWarning` |

### Manual Upgrade Commands

| Action | Command |
|--------|---------|
| Check available versions | `curl -s https://update.rke2.io/v1-release/channels \| jq .data` |
| Upgrade server binary | `curl -sfL https://get.rke2.io \| INSTALL_RKE2_CHANNEL=v1.34 sh -` |
| Upgrade agent binary | `curl -sfL https://get.rke2.io \| INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=v1.34 sh -` |
| Restart server | `systemctl restart rke2-server` |
| Restart agent | `systemctl restart rke2-agent` |

### System Upgrade Controller Commands

| Action | Command |
|--------|---------|
| Install SUC | `kubectl apply -f .../crd.yaml -f .../system-upgrade-controller.yaml` |
| Check plans | `kubectl get plans -n system-upgrade` |
| Watch upgrade jobs | `kubectl get jobs -n system-upgrade -w` |
| Remove SUC | Delete plans, then controller, then CRD (see Cleanup section) |

## Common Errors (Searchable)

```
x509: certificate has expired or is not yet valid
```
**Cause:** RKE2 certificates have expired. The service ran for more than 245 days without a restart, missing the 120-day auto-renewal window. **Fix:** Stop the service, run `rke2 certificate rotate`, verify with `rke2 certificate check --output table`, then start the service.

```
CertificateExpirationWarning
```
**Cause:** Kubernetes event indicating a certificate is within 120 days of expiry. **Fix:** Schedule a maintenance window to restart RKE2 (triggers auto-renewal) or manually rotate certificates.

```
Unable to connect to the server: x509: certificate signed by unknown authority
```
**Cause:** kubeconfig contains an old client certificate after rotation. **Fix:** Copy the updated kubeconfig: `cp /etc/rancher/rke2/rke2.yaml ~/.kube/config`.

```
error: error upgrading connection: error dialing backend: x509: certificate is valid for <old-names>, not <new-name>
```
**Cause:** SAN mismatch after node hostname or IP change. The certificate was issued for different Subject Alternative Names. **Fix:** Rotate certificates to regenerate with current node identity.

```
level=error msg="unable to start controller: tls: failed to find any PEM data in certificate input"
```
**Cause:** Certificate file is empty or corrupted, possibly from a failed rotation. **Fix:** Check the timestamped backup directory under `/var/lib/rancher/rke2/server/tls-*/`, restore the previous certificates, and retry the rotation.

```
Error from server (NotFound): plans.upgrade.cattle.io "server-plan" not found
```
**Cause:** The Plan CRD is not installed or the plan was not applied. **Fix:** Ensure the CRD is installed with `kubectl get crd plans.upgrade.cattle.io`, then apply the plan YAML.

```
Job has reached the specified backoff limit
```
**Cause:** The upgrade job on a node failed repeatedly. **Fix:** Check the job pod logs with `kubectl logs -n system-upgrade <pod-name>`. Common issues: node disk full, network issues pulling the upgrade image, or insufficient permissions.

```
node "<node-name>" already has a newer version
```
**Cause:** The plan targets a node that already runs a version equal to or newer than the channel's resolved version. **Fix:** No action needed; the controller skips nodes that are already at or above the target version.

```
error: unable to drain node: cannot evict pod as it would violate the pod's disruption budget
```
**Cause:** A PodDisruptionBudget prevents draining the node during upgrade. **Fix:** Audit PDBs with `kubectl get pdb -A`, adjust `maxUnavailable` if set to 0, or temporarily delete the blocking PDB for the upgrade window.

```
rke2-server.service: Failed with result 'exit-code'
```
**Cause:** RKE2 server failed to start after upgrade or certificate rotation. **Fix:** Check full logs with `journalctl -xeu rke2-server`. Common causes: port conflicts, corrupted certificates, or incompatible configuration after version upgrade.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Upgrading agents before servers | Agent kubelet version newer than API server; unsupported skew, potential API incompatibilities |
| Running `rke2 certificate rotate` without stopping the service first | Rotation may fail or produce inconsistent state; always `systemctl stop rke2-server` first |
| Not copying updated kubeconfig after certificate rotation | `kubectl` commands fail with x509 errors because the local kubeconfig has old client certificates |
| Letting the cluster run 245+ days without a service restart | Certificates pass the 120-day auto-renewal window and expire at 365 days, causing cluster outage |
| Applying agent-plan without server-plan | Workers upgrade but control plane stays on the old version; reversed version skew breaks the cluster |
| Setting SUC agent-plan concurrency too high | Too many workers drain simultaneously; workloads have nowhere to schedule, causing application downtime |
| Not checking `rke2 certificate check` after rotation | Rotation may have partially failed; unverified certificates lead to surprise outages |
| Skipping pre-upgrade monitoring setup | No visibility into whether the upgrade caused application downtime; problems discovered too late |
| Forgetting tolerations on server-plan | Upgrade pods cannot schedule on control plane nodes that have taints; upgrade never starts |
| Not cleaning up the System Upgrade Controller after upgrade | Leftover controller may trigger unintended upgrades when a new version appears in the channel |
| Upgrading RKE2 without draining workers in production | Pods on the node are abruptly terminated during restart; causes brief application unavailability |
| Not verifying etcd health before starting upgrade | Starting an upgrade with a degraded etcd cluster risks total data loss |
