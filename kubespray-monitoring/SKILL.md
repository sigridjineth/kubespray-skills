---
name: kubespray-monitoring
description: Use when setting up Prometheus, Grafana, and Alertmanager monitoring on Kubespray clusters, configuring etcd metrics collection, deploying NFS storage provisioners, or importing Grafana dashboards.
---

# Kubespray Cluster Monitoring

## Overview

Production monitoring for Kubespray-deployed clusters uses a three-layer stack: NFS provisioner for persistent storage, kube-prometheus-stack for metrics collection and alerting, and etcd metrics exposure for cluster health visibility.

**Core principle:** Monitoring must cover all cluster layers - infrastructure (node-exporter), Kubernetes components (API server, scheduler, controller-manager), etcd (leader, WAL, peer latency), and workloads (pod metrics via kube-state-metrics).

## When to Use

- Setting up Prometheus, Grafana, and Alertmanager on Kubespray clusters
- Deploying NFS storage provisioner for monitoring persistence
- Importing Grafana dashboards (community or custom)
- Enabling etcd metrics collection
- Writing PromQL queries for cluster health

**Not for:** Initial cluster deployment (use kubespray-deployment), HA configuration (use kubespray-ha-configuration), cluster upgrades (use kubespray-operations), troubleshooting failures (use kubespray-troubleshooting)

## NFS Subdir External Provisioner

kube-prometheus-stack requires persistent volumes for Prometheus and Grafana data. Without a StorageClass, PVCs stay in `Pending` state and pods never start. NFS subdir external provisioner creates a default StorageClass backed by an existing NFS server.

### Installation

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.10.10 \
  --set nfs.path=/srv/nfs/kubedata \
  --set storageClass.defaultClass=true
```

**Parameters:**
- `nfs.server`: IP of NFS server (often the HAProxy/load-balancer node)
- `nfs.path`: Exported NFS path (must exist and be exported via `/etc/exports`)
- `storageClass.defaultClass=true`: Makes this the default StorageClass so PVCs without explicit `storageClassName` bind automatically

### Verification

```bash
# Check StorageClass is default
kubectl get storageclass
# NAME                   PROVISIONER                                     AGE
# nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   1m

# Check provisioner pod is running
kubectl get pods -l app=nfs-subdir-external-provisioner
```

## kube-prometheus-stack Installation

### Helm Repository Setup

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update
```

### Custom Values File

Create `custom-values.yaml` with the following configuration:

```yaml
# custom-values.yaml
prometheus:
  prometheusSpec:
    # Discover all ServiceMonitors regardless of labels
    serviceMonitorSelectorNilUsesHelmValues: false
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    # Scrape HAProxy and etcd endpoints
    additionalScrapeConfigs:
      - job_name: 'haproxy'
        static_configs:
          - targets: ['192.168.10.10:8405']
      - job_name: 'etcd'
        static_configs:
          - targets:
              - '192.168.10.11:2381'
              - '192.168.10.12:2381'
              - '192.168.10.13:2381'
  service:
    type: NodePort
    nodePort: 30001

grafana:
  adminPassword: prom-operator
  service:
    type: NodePort
    nodePort: 30002
  persistence:
    enabled: true
    size: 5Gi
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL

nodeExporter:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"

alertmanager:
  service:
    type: NodePort
    nodePort: 30003
```

**Key settings explained:**
- `serviceMonitorSelectorNilUsesHelmValues: false`: Without this, Prometheus only discovers ServiceMonitors with helm-specific labels, missing custom monitors
- `additionalScrapeConfigs`: Static targets for infrastructure not running inside Kubernetes (HAProxy on port 8405, etcd on port 2381)
- `nodeExporter.tolerations`: Required so node-exporter DaemonSet runs on control plane nodes (which have `NoSchedule` taint), giving full cluster coverage
- `sidecar.dashboards.searchNamespace: ALL`: Grafana sidecar watches all namespaces for ConfigMaps with the `grafana_dashboard: "1"` label

### Installing the Stack

```bash
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 80.13.3 \
  -f custom-values.yaml
```

### Verification

```bash
# All pods should be Running (takes 2-5 minutes)
kubectl get pods -n monitoring

# Expected pods:
# - alertmanager-kube-prometheus-stack-alertmanager-0
# - kube-prometheus-stack-grafana-*
# - kube-prometheus-stack-kube-state-metrics-*
# - kube-prometheus-stack-operator-*
# - kube-prometheus-stack-prometheus-node-exporter-* (one per node)
# - prometheus-kube-prometheus-stack-prometheus-0

# Verify node-exporter runs on ALL nodes including control planes
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter -o wide
# Should show 5 pods (3 control plane + 2 worker) if 5-node cluster
```

**Access URLs (replace NODE_IP with any node IP):**
- Prometheus: `http://NODE_IP:30001`
- Grafana: `http://NODE_IP:30002` (admin / prom-operator)
- Alertmanager: `http://NODE_IP:30003`

## Grafana Dashboard Import

### Via ConfigMap Sidecar

The Grafana sidecar automatically loads dashboards from ConfigMaps labeled with `grafana_dashboard: "1"`. This is the recommended approach because dashboards survive pod restarts.

#### Download Community Dashboards

```bash
# Kubernetes Monitoring (Dashboard ID: 12693)
curl -s https://grafana.com/api/dashboards/12693/revisions/latest/download \
  -o /tmp/kubernetes-monitoring.json

# Node Exporter Full (Dashboard ID: 15661)
curl -s https://grafana.com/api/dashboards/15661/revisions/latest/download \
  -o /tmp/node-exporter-full.json
```

#### Create ConfigMaps

```bash
kubectl create configmap grafana-dashboard-kubernetes-monitoring \
  --from-file=kubernetes-monitoring.json=/tmp/kubernetes-monitoring.json \
  -n monitoring

kubectl label configmap grafana-dashboard-kubernetes-monitoring \
  grafana_dashboard="1" -n monitoring

kubectl create configmap grafana-dashboard-node-exporter-full \
  --from-file=node-exporter-full.json=/tmp/node-exporter-full.json \
  -n monitoring

kubectl label configmap grafana-dashboard-node-exporter-full \
  grafana_dashboard="1" -n monitoring
```

Dashboards appear in Grafana within 60 seconds (sidecar polling interval).

### Custom API Server Dashboard

Create a ConfigMap with an inline JSON dashboard for API server monitoring:

```yaml
# api-server-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-api-server
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  api-server-dashboard.json: |
    {
      "annotations": { "list": [] },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "panels": [
        {
          "title": "API Server Request Rate",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
          "targets": [
            {
              "expr": "sum(rate(apiserver_request_total[5m])) by (verb)",
              "legendFormat": "{{ verb }}"
            }
          ]
        },
        {
          "title": "API Server Latency (99th Percentile)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (verb, le))",
              "legendFormat": "{{ verb }}"
            }
          ]
        },
        {
          "title": "API Server Error Rate",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
          "targets": [
            {
              "expr": "sum(rate(apiserver_request_total{code=~\"5..\"}[5m])) by (verb)",
              "legendFormat": "{{ verb }} 5xx"
            }
          ]
        },
        {
          "title": "API Server Inflight Requests",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 },
          "targets": [
            {
              "expr": "sum(apiserver_current_inflight_requests) by (request_kind)",
              "legendFormat": "{{ request_kind }}"
            }
          ]
        }
      ],
      "schemaVersion": 39,
      "tags": ["kubernetes", "api-server"],
      "templating": { "list": [] },
      "time": { "from": "now-1h", "to": "now" },
      "title": "API Server Overview",
      "uid": "api-server-overview"
    }
```

```bash
kubectl apply -f api-server-dashboard.yaml
```

## Enabling etcd Metrics

By default, etcd in Kubespray only listens on HTTPS port 2379 with mTLS. Exposing a separate HTTP metrics endpoint avoids certificate complexity for Prometheus scraping.

### Configuration

Edit `inventory/mycluster/group_vars/etcd.yml`:

```yaml
# etcd.yml
etcd_metrics: true
etcd_listen_metrics_urls: "http://0.0.0.0:2381"
```

**Why these values:**
- **HTTP not HTTPS:** The metrics endpoint does not expose sensitive data. Using HTTP avoids needing to distribute etcd TLS certificates to Prometheus, which would require mounting secrets and managing rotation.
- **0.0.0.0:** Binds to all interfaces so Prometheus (running on worker nodes) can reach etcd (running on control plane nodes) across the network. Using `127.0.0.1` would only allow local access.
- **Port 2381:** Avoids conflict with the main etcd client port (2379) and peer port (2380). Port 2381 is the community convention for etcd metrics.

### Applying with Kubespray

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --tags etcd --limit etcd -b
```

This performs a rolling restart of etcd members one at a time. Kubespray automatically:
1. Creates a backup in `/var/backups/` before modifying each member
2. Updates the etcd configuration
3. Restarts the etcd service
4. Waits for the member to rejoin the cluster before proceeding to the next

**Important:** The `--limit etcd` flag restricts execution to etcd nodes only. The `--tags etcd` flag runs only etcd-related tasks, not the full cluster playbook.

### Verification

```bash
# Check metrics endpoint on each etcd member
curl http://192.168.10.11:2381/metrics | head -20
curl http://192.168.10.12:2381/metrics | head -20
curl http://192.168.10.13:2381/metrics | head -20

# Look for key metrics in output:
# etcd_server_has_leader 1
# etcd_disk_wal_fsync_duration_seconds_bucket{...}
# etcd_network_peer_round_trip_time_seconds_bucket{...}
```

### Prometheus Target Verification

After enabling etcd metrics, verify Prometheus is scraping them:

1. Open Prometheus UI: `http://NODE_IP:30001`
2. Navigate to **Status -> Targets**
3. Find the `etcd` job - all 3 targets should show state **UP**

**Troubleshooting if targets show DOWN:**
- **Firewall:** Ensure port 2381 is open between worker nodes and control plane nodes: `sudo iptables -L -n | grep 2381`
- **Network:** Test connectivity from a worker node: `curl http://192.168.10.11:2381/metrics`
- **etcd process:** Verify etcd is listening on 2381: `ss -tlnp | grep 2381` on the control plane node
- **Configuration:** Check etcd process flags: `ps aux | grep etcd | grep listen-metrics` should show `--listen-metrics-urls=http://0.0.0.0:2381`

## Sample PromQL Queries

### etcd Health

```promql
# Leader status (1 = has leader, 0 = no leader - CRITICAL)
etcd_server_has_leader

# WAL fsync 99th percentile (should be < 10ms for SSD)
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# Database size in bytes (alert if > 6GB, etcd limit is 8GB)
etcd_mvcc_db_total_size_in_bytes

# Peer round-trip time 99th percentile (should be < 50ms)
histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[5m]))

# Leader change rate (should be 0 in steady state, > 0 indicates instability)
rate(etcd_server_leader_changes_seen_total[1h])
```

### Cluster Health

```promql
# Node CPU usage
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Pod restart rate (possible crash loops)
rate(kube_pod_container_status_restarts_total[15m]) > 0

# API server request error rate
sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))
```

## Complete Target Summary

After full setup, Prometheus should show these targets under **Status -> Targets**:

| Target | Endpoints | Expected Count | Source |
|--------|-----------|----------------|--------|
| kubernetes-apiservers | https://CP_IP:6443 | 3/3 | Service discovery |
| kubernetes-nodes | https://NODE_IP:10250 | 5/5 | Service discovery |
| node-exporter | http://NODE_IP:9100 | 5/5 | ServiceMonitor |
| kube-state-metrics | http://POD_IP:8080 | 1/1 | ServiceMonitor |
| haproxy | http://192.168.10.10:8405 | 1/1 | additionalScrapeConfigs |
| etcd | http://CP_IP:2381 | 3/3 | additionalScrapeConfigs |

Total: 18 targets across 6 jobs for a 5-node cluster (3 control plane + 2 worker).

## Common Errors (Searchable)

```
prometheus-kube-prometheus-stack-prometheus-0   0/2     Pending   0          5m
```
**Cause:** No StorageClass available, PVC cannot bind. **Fix:** Install NFS provisioner first, verify `kubectl get storageclass` shows a default class.

```
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```
**Cause:** Helm release already exists. **Fix:** Use `helm upgrade` instead of `helm install`, or `helm uninstall kube-prometheus-stack -n monitoring` first.

```
Get "http://192.168.10.11:2381/metrics": dial tcp 192.168.10.11:2381: connect: connection refused
```
**Cause:** etcd not listening on metrics port. **Fix:** Verify `etcd_metrics: true` in etcd.yml, re-run playbook with `--tags etcd`.

```
level=warn msg="Error on ingesting samples that are too old or are too far into the future"
```
**Cause:** Clock skew between nodes. **Fix:** Enable NTP on all nodes: `group_vars/all/all.yml` set `ntp_enabled: true`.

```
Grafana dashboard shows "No data"
```
**Cause:** Datasource misconfigured or Prometheus not scraping target. **Fix:** Check Prometheus targets page first, then verify Grafana datasource URL matches Prometheus service (`http://kube-prometheus-stack-prometheus.monitoring:9090`).

```
kube-prometheus-stack-prometheus-node-exporter   CrashLoopBackOff
```
**Cause:** Port 9100 already in use on node (existing node_exporter installation). **Fix:** Remove existing node_exporter from host or change port in values.yaml.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Installing kube-prometheus-stack without StorageClass | Prometheus and Grafana pods stuck in Pending |
| Missing `serviceMonitorSelectorNilUsesHelmValues: false` | Prometheus ignores custom ServiceMonitors |
| etcd metrics over HTTPS without cert distribution | Prometheus cannot scrape etcd, targets show DOWN |
| No node-exporter tolerations for control plane | Missing metrics for 3 out of 5 nodes |
| Forgetting `searchNamespace: ALL` on Grafana sidecar | Dashboards in other namespaces not loaded |
| Using `helm install` for updates | Fails with "name already in use", use `helm upgrade` |
| Not opening port 2381 in firewall | etcd targets show DOWN in Prometheus |
| Skipping `--create-namespace` on first install | Fails with namespace "monitoring" not found |
