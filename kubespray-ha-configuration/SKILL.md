---
name: kubespray-ha-configuration
description: Use when setting up high availability Kubernetes clusters, configuring multiple control plane nodes, etcd quorum sizing, or implementing load balancing for API server access.
---

# Kubespray HA Configuration

## Overview

High availability in Kubernetes requires redundant control plane nodes and etcd members. Kubespray automates HA setup but understanding the architecture is essential for proper sizing and troubleshooting.

**Core principle:** API servers are active-active (all handle requests). Controller-manager and scheduler use leader election (one active, others standby). etcd requires quorum (majority must agree).

## When to Use

- Deploying production clusters requiring uptime guarantees
- Sizing etcd clusters (why odd numbers matter)
- Configuring load balancing for API server
- Understanding failover behavior

**Not for:** Single-node deployments (use kubespray-deployment), troubleshooting HA failures (use kubespray-troubleshooting)

## HA Architecture

```
                        +-----------------------+
                        |   External Clients    |
                        +-----------+-----------+
                                    |
                        +-----------v-----------+
                        | External LB (optional)|
                        | HAProxy / Cloud LB    |
                        +-----------+-----------+
                                    |
                +-------------------+-------------------+
                |                   |                   |
      +---------v--------+ +-------v---------+ +-------v---------+
      |   k8s-ctr1       | |   k8s-ctr2      | |   k8s-ctr3      |
      | API Server  :6443| | API Server :6443| | API Server :6443|
      | Ctrl-Mgr (lead)  | | Ctrl-Mgr (stby) | | Ctrl-Mgr (stby) |
      | Scheduler (lead) | | Scheduler (stby) | | Scheduler (stby) |
      | etcd             | | etcd            | | etcd            |
      +------------------+ +-----------------+ +-----------------+
                +-------------------+-------------------+
                |                   |
      +---------v--------+ +-------v---------+
      |   k8s-w1         | |   k8s-w2        |
      | nginx-proxy      | | nginx-proxy     |
      | (localhost:6443)  | | (localhost:6443)|
      | kubelet, kube-prx | | kubelet, kube-prx|
      +------------------+ +-----------------+
```

```
  API Server:        Active-Active (all handle requests)
  Controller-Mgr:    Active-Standby (leader election)
  Scheduler:         Active-Standby (leader election)
  etcd:              Raft consensus (quorum required)
```

## etcd Quorum Sizing

| Nodes | Quorum | Tolerated Failures |
|-------|--------|-------------------|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Always use odd numbers.** With 4 nodes, quorum is 3 - you can still only lose 1 node (same as 3 nodes). The extra node adds complexity without fault tolerance.

## Inventory for HA Cluster

```ini
[all]
k8s-ctr1 ansible_host=192.168.10.11 ip=192.168.10.11
k8s-ctr2 ansible_host=192.168.10.12 ip=192.168.10.12
k8s-ctr3 ansible_host=192.168.10.13 ip=192.168.10.13
k8s-w1 ansible_host=192.168.10.21 ip=192.168.10.21
k8s-w2 ansible_host=192.168.10.22 ip=192.168.10.22

[kube_control_plane]
k8s-ctr1
k8s-ctr2
k8s-ctr3

[etcd]
k8s-ctr1
k8s-ctr2
k8s-ctr3

[kube_node]
k8s-w1
k8s-w2

[k8s_cluster:children]
kube_control_plane
kube_node
```

This is **stacked etcd** - etcd runs on control plane nodes. For **external etcd**, use separate nodes in `[etcd]` group.

## Load Balancing for the API Server

Kubespray provides three approaches to load balancing API server traffic. Each offers different trade-offs between simplicity, resilience, and external accessibility.

---

### Case 1: Client-Side LB with NGINX Static Pods (Default)

This is the default configuration. Kubespray deploys an NGINX reverse proxy as a static pod on **every worker node**. Each worker independently load-balances to all control plane nodes.

```
                 +-------------------+-------------------+
                 |                   |                   |
       +---------v--------+ +-------v---------+ +-------v---------+
       |   k8s-ctr1       | |   k8s-ctr2      | |   k8s-ctr3      |
       | apiserver :6443  | | apiserver :6443  | | apiserver :6443  |
       +--^-------^-------+ +--^-------^------+ +--^-------^------+
          |       |            |       |           |       |
          |  +----+------------+-------+-----------+       |
          |  |    |            |                           |
       +--+--+----+--------+ +----------------------------+--+
       |  nginx-proxy      | |  nginx-proxy                  |
       |  127.0.0.1:6443   | |  127.0.0.1:6443               |
       |  (static pod)     | |  (static pod)                 |
       +-------------------+ +-------------------------------+
       |  k8s-w1           | |  k8s-w2                       |
       |  kubelet -> localhost:6443                           |
       +-------------------+ +-------------------------------+
```

**Configuration** (`group_vars/all/all.yml`):
```yaml
# This is the default - explicitly shown for clarity
loadbalancer_apiserver_localhost: true
loadbalancer_apiserver_type: nginx  # or haproxy
```

**How it works:**

Kubespray generates an NGINX configuration on each worker node that performs L4 (TCP) proxying to all control plane API servers:

```nginx
# /etc/nginx/nginx.conf (on each worker node)
error_log stderr notice;

worker_processes 1;
worker_rlimit_nofile 130048;
worker_shutdown_timeout 10s;

events {
  multi_accept on;
  use epoll;
  worker_connections 16384;
}

stream {
  upstream kube_apiserver {
    least_conn;
    server 192.168.10.11:6443;
    server 192.168.10.12:6443;
    server 192.168.10.13:6443;
  }

  server {
    listen 127.0.0.1:6443;
    proxy_pass kube_apiserver;
    proxy_timeout 10m;
    proxy_connect_timeout 1s;
  }
}
```

Key details about the NGINX config:
- Uses `stream` block (L4 TCP proxy), **not** `http` block -- this is raw TCP passthrough, not HTTP reverse proxy
- `least_conn` distributes connections to the server with fewest active connections
- `proxy_connect_timeout 1s` means fast failover: if a CP node is down, NGINX moves on within 1 second
- `proxy_timeout 10m` allows long-running watch streams (kubectl logs -f, watch operations)
- Listens only on `127.0.0.1:6443` (loopback), not externally accessible

The static pod manifest deployed by Kubespray:

```yaml
# /etc/kubernetes/manifests/nginx-proxy.yaml (on each worker)
apiVersion: v1
kind: Pod
metadata:
  name: nginx-proxy
  namespace: kube-system
  labels:
    k8s-app: kube-nginx
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  priorityClassName: system-node-critical
  containers:
  - name: nginx-proxy
    image: docker.io/library/nginx:1.27.2-alpine
    resources:
      requests:
        cpu: 25m
        memory: 32M
    volumeMounts:
    - mountPath: /etc/nginx
      name: etc-nginx
      readOnly: true
    livenessProbe:
      httpGet: null
      tcpSocket:
        host: 127.0.0.1
        port: 6443
  volumes:
  - name: etc-nginx
    hostPath:
      path: /etc/nginx
      type: Directory
```

Key details about the static pod:
- `hostNetwork: true` -- the pod uses the host's network namespace so it can bind to `127.0.0.1:6443`
- `system-node-critical` priority -- ensures the proxy is never evicted
- Control plane nodes do **not** run nginx-proxy (they have a local apiserver already on port 6443)

**Kubelet configuration on workers:**
```yaml
# /etc/kubernetes/kubelet.conf (on each worker)
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://localhost:6443
    # ...
```

All kubelet traffic goes to `localhost:6443` -> nginx-proxy -> one of the CP nodes.

**Pros:**
- No external infrastructure required
- Fast failover (1 second connect timeout)
- Each worker is independently resilient -- one worker's proxy failure does not affect others
- Zero additional network hops for internal traffic

**Cons:**
- No single external endpoint for kubectl access outside the cluster
- NGINX config on workers must be updated when CP nodes are added/removed (Kubespray handles this automatically)
- External clients must connect directly to a specific CP node's IP

#### Failure Simulation: Case 1

**Step 1: Verify baseline**
```bash
# On a worker node, check nginx-proxy is running
crictl ps | grep nginx-proxy

# Verify NGINX upstream config
cat /etc/nginx/nginx.conf
# Should show all 3 CP servers in upstream block

# Test connectivity
curl -k https://localhost:6443/healthz
# Expected: ok
```

**Step 2: Simulate CP node failure**
```bash
# Stop API server on k8s-ctr1
ssh k8s-ctr1 "systemctl stop kubelet"

# Immediately test from a worker node - should succeed within 1s failover
curl -k https://localhost:6443/healthz
# Expected: ok (NGINX fails over to ctr2 or ctr3 within 1s)

# Verify kubectl still works
kubectl get nodes
# k8s-ctr1 shows NotReady, but commands succeed
```

**Step 3: Simulate multiple CP failures (quorum intact)**
```bash
# With 3 CP nodes, stop 1 (quorum: 2 of 3 remain)
ssh k8s-ctr1 "systemctl stop kubelet && systemctl stop etcd"

# API should still work (2 CP nodes + 2 etcd = quorum)
kubectl get nodes
kubectl create configmap failover-test --from-literal=test=pass

# Stop a second CP (quorum LOST: only 1 of 3 etcd)
ssh k8s-ctr2 "systemctl stop kubelet && systemctl stop etcd"

# API server on ctr3 is up but etcd has no quorum
# Reads may work but writes WILL FAIL:
kubectl create configmap should-fail --from-literal=test=fail
# Expected: etcdserver: request timed out
```

**Step 4: Restore**
```bash
ssh k8s-ctr2 "systemctl start etcd && systemctl start kubelet"
ssh k8s-ctr1 "systemctl start etcd && systemctl start kubelet"

# Verify full recovery
kubectl get nodes
# All nodes should return to Ready
```

---

### Case 2: External LB (HAProxy) + Client-Side LB (Recommended for Production)

This combines an external HAProxy load balancer for outside access with NGINX static pods on workers for internal resilience. Two independent failover paths operate simultaneously.

```
     +-----------------------+
     |   External Clients    |
     |  kubectl, CI/CD, etc  |
     +-----------+-----------+
                 |
     +-----------v-----------+
     |   HAProxy (admin-lb)  |
     | 192.168.10.10:6443    |
     | k8s-api-srv.admin-lb  |
     +-----------+-----------+
                 |
     +-----------v-----+----------+-----------+
     |                 |                      |
  +--v---------+  +----v------+  +-----------v+
  | k8s-ctr1   |  | k8s-ctr2 |  | k8s-ctr3   |
  | api :6443  |  | api :6443|  | api :6443  |
  +--^---^-----+  +--^---^---+  +--^---^-----+
     |   |           |   |        |   |
     |   +--------+--+---+--------+   |
     |            |  |                 |
  +--+-----------+|  +-----------------+--+
  | nginx-proxy  ||  | nginx-proxy       |
  | 127.0.0.1    ||  | 127.0.0.1        |
  | :6443        ||  | :6443            |
  +--------------+|  +-------------------+
  | k8s-w1       ||  | k8s-w2           |
  +--------------+   +-------------------+

  External path:  Client -> HAProxy -> CP node
  Internal path:  Worker -> nginx-proxy (localhost) -> CP node
```

**Configuration:**

Workers keep using nginx-proxy (default behavior). You add an external HAProxy separately.

`group_vars/all/all.yml` -- keep the default:
```yaml
loadbalancer_apiserver_localhost: true  # keep nginx-proxy on workers
```

**CRITICAL: Update certificate SANs** to include the HAProxy address. Without this, TLS connections through HAProxy will fail with certificate errors.

`group_vars/k8s_cluster/k8s-cluster.yml`:
```yaml
supplementary_addresses_in_ssl_keys:
  - 192.168.10.10
  - k8s-api-srv.admin-lb.com
```

**Deploy/update certificates:**
```bash
# Regenerate API server certificates with new SANs
ansible-playbook cluster.yml \
  --tags "control-plane" \
  --limit kube_control_plane
```

**Verify certificate SANs include the HAProxy address:**
```bash
openssl x509 \
  -in /etc/kubernetes/ssl/apiserver.crt \
  -noout -text | grep -A 20 "Subject Alternative Name"

# Expected output should include:
#   IP Address:192.168.10.10
#   DNS:k8s-api-srv.admin-lb.com
```

**HAProxy configuration** (on admin-lb host `192.168.10.10`):
```
# /etc/haproxy/haproxy.cfg
frontend k8s_api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s_api_servers

backend k8s_api_servers
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-ctr1 192.168.10.11:6443 check fall 3 rise 2
    server k8s-ctr2 192.168.10.12:6443 check fall 3 rise 2
    server k8s-ctr3 192.168.10.13:6443 check fall 3 rise 2
```

**Update kubeconfig for external access:**
```bash
kubectl config set-cluster cluster.local \
  --server=https://k8s-api-srv.admin-lb.com:6443
```

**Two independent failover mechanisms:**
- **External clients** (kubectl from workstation, CI/CD): go through HAProxy, which health-checks CP nodes and removes failed ones
- **Internal components** (kubelet, kube-proxy on workers): use nginx-proxy on localhost, which independently fails over within 1 second

**Pros:**
- Single stable endpoint for external access (HAProxy VIP or DNS)
- Internal cluster traffic remains independently resilient via nginx-proxy
- Losing HAProxy does not affect internal cluster operations
- Losing nginx-proxy on one worker does not affect other workers or external access
- Best resilience profile: two independent failure domains

**Cons:**
- Requires additional infrastructure for HAProxy (and ideally keepalived for HAProxy HA)
- Must remember to update `supplementary_addresses_in_ssl_keys` for certificate SANs
- Two load balancing layers to monitor/maintain

#### Failure Simulation: Case 2

**Step 1: Verify both paths**
```bash
# External path (via HAProxy)
curl -k https://192.168.10.10:6443/healthz
# Expected: ok

# Internal path (from a worker, via nginx-proxy)
ssh k8s-w1 "curl -k https://localhost:6443/healthz"
# Expected: ok
```

**Step 2: Simulate HAProxy failure**
```bash
# Stop HAProxy
ssh admin-lb "systemctl stop haproxy"

# External access FAILS
curl -k https://192.168.10.10:6443/healthz
# Expected: connection refused

# Internal cluster UNAFFECTED
kubectl --kubeconfig=/etc/kubernetes/kubelet.conf get nodes
# Works fine - workers use nginx-proxy, not HAProxy
ssh k8s-w1 "curl -k https://localhost:6443/healthz"
# Expected: ok

# Restore HAProxy
ssh admin-lb "systemctl start haproxy"
```

**Step 3: Simulate CP node failure with both paths**
```bash
# Stop k8s-ctr1
ssh k8s-ctr1 "systemctl stop kubelet"

# HAProxy detects failure (fall 3 = after 3 failed checks) and removes ctr1
# External access continues through ctr2/ctr3
curl -k https://192.168.10.10:6443/healthz
# Expected: ok

# nginx-proxy on workers also fails over (within 1s)
ssh k8s-w1 "curl -k https://localhost:6443/healthz"
# Expected: ok

# Restore
ssh k8s-ctr1 "systemctl start kubelet"
```

**Step 4: Verify certificate SANs are correct**
```bash
# If you see this error through HAProxy:
#   Unable to connect to the server: x509: certificate is valid for
#   192.168.10.11, 192.168.10.12, 192.168.10.13, not 192.168.10.10

# The SAN was not added. Fix:
# 1. Add to supplementary_addresses_in_ssl_keys
# 2. Re-run: ansible-playbook cluster.yml --tags "control-plane" --limit kube_control_plane
# 3. Verify: openssl x509 -in /etc/kubernetes/ssl/apiserver.crt -noout -text | grep -A 20 "Subject Alternative Name"
```

---

### Case 3: External LB as Single Endpoint (All Components)

All traffic -- both external clients and internal cluster components (kubelet, kube-proxy) -- goes through a single external load balancer. NGINX static pods are removed from workers.

```
     +-----------------------+
     |   External Clients    |
     +-----------+-----------+
                 |
     +-----------v-----------+
     |   HAProxy (admin-lb)  |
     | 192.168.10.10:6443    |
     | k8s-api-srv.admin-lb  |
     +-----------+-----------+
                 |
     +-----------v-----+----------+-----------+
     |                 |                      |
  +--v---------+  +----v------+  +-----------v+
  | k8s-ctr1   |  | k8s-ctr2 |  | k8s-ctr3   |
  | api :6443  |  | api :6443|  | api :6443  |
  +------------+  +----------+  +------------+
                       ^
                       |
     +-----------------+-----------------+
     |                                   |
  +--+-------------+  +-----------------+-+
  | k8s-w1         |  | k8s-w2           |
  | NO nginx-proxy |  | NO nginx-proxy   |
  | kubelet -> HAProxy:6443              |
  +----------------+  +------------------+

  ALL traffic flows through HAProxy
  nginx-proxy is NOT deployed on workers
```

**Configuration** (`group_vars/all/all.yml`):
```yaml
# Disable client-side LB -- removes nginx-proxy from workers
loadbalancer_apiserver_localhost: false

# Set the external LB as the API server endpoint
apiserver_loadbalancer_domain_name: "k8s-api-srv.admin-lb.com"
loadbalancer_apiserver:
  address: 192.168.10.10
  port: 6443
```

`group_vars/k8s_cluster/k8s-cluster.yml`:
```yaml
supplementary_addresses_in_ssl_keys:
  - 192.168.10.10
  - k8s-api-srv.admin-lb.com
```

**WARNING: This is a DISRUPTIVE change.** If switching from Case 1 or Case 2 to Case 3 on an existing cluster, this requires a full `cluster.yml` run (not just `--tags`). All kubelets will be reconfigured to point to the external LB.

```bash
# Full cluster playbook run required
ansible-playbook cluster.yml
```

**Post-deployment verification:**
```bash
# 1. Verify nginx-proxy pods are GONE from workers
ssh k8s-w1 "crictl ps | grep nginx-proxy"
# Expected: no output (nginx-proxy removed)

# 2. Verify kubelet.conf on workers points to external LB
ssh k8s-w1 "grep server /etc/kubernetes/kubelet.conf"
# Expected: server: https://k8s-api-srv.admin-lb.com:6443

# 3. Verify kubelet.conf on CP nodes also points to external LB
ssh k8s-ctr1 "grep server /etc/kubernetes/kubelet.conf"
# Expected: server: https://k8s-api-srv.admin-lb.com:6443

# 4. Verify kube-proxy configmap points to external LB
kubectl -n kube-system get configmap kube-proxy -o yaml | grep server
# Expected: server: https://k8s-api-srv.admin-lb.com:6443
```

**WARNING: HAProxy becomes a Single Point of Failure (SPOF).** If HAProxy goes down, the entire cluster loses API access -- including internal components like kubelet and kube-proxy. Mitigate with:

- **keepalived + VRRP:** Run two HAProxy instances with a floating VIP using VRRP. If the primary fails, the secondary takes over the VIP automatically.
- **Cloud managed LB:** Use AWS NLB, GCP LB, or Azure LB which provide built-in HA.
- **DNS round-robin with health checks:** Multiple HAProxy instances behind DNS with health-check-based failover (slower failover than VRRP).

**Pros:**
- Single, consistent endpoint for ALL traffic (internal and external)
- Simpler mental model -- one path to the API server
- Easier to monitor and audit (one chokepoint for all API traffic)
- No need to manage nginx-proxy on workers

**Cons:**
- HAProxy is a SPOF unless made highly available (keepalived, cloud LB)
- All cluster components depend on external network path to LB
- Network partition between workers and LB takes down the entire cluster (not just external access)
- Higher latency for internal traffic (extra hop through LB)
- Full `cluster.yml` run required to switch to this mode

#### Failure Simulation: Case 3

**Step 1: Verify single path**
```bash
# All access goes through HAProxy
curl -k https://192.168.10.10:6443/healthz
# Expected: ok

# Workers use the same path
ssh k8s-w1 "curl -k https://k8s-api-srv.admin-lb.com:6443/healthz"
# Expected: ok

# Confirm no nginx-proxy
ssh k8s-w1 "crictl ps | grep nginx-proxy"
# Expected: empty (no nginx-proxy)
```

**Step 2: Simulate CP node failure**
```bash
# Stop k8s-ctr1
ssh k8s-ctr1 "systemctl stop kubelet"

# HAProxy removes ctr1 from rotation
# All traffic (external AND internal) continues through ctr2/ctr3
curl -k https://192.168.10.10:6443/healthz
# Expected: ok

kubectl get nodes
# ctr1 shows NotReady, everything else works
```

**Step 3: Simulate HAProxy failure (CRITICAL)**
```bash
# Stop HAProxy
ssh admin-lb "systemctl stop haproxy"

# EVERYTHING breaks - external AND internal
curl -k https://192.168.10.10:6443/healthz
# Expected: connection refused

# Workers cannot reach API server
ssh k8s-w1 "curl -k https://k8s-api-srv.admin-lb.com:6443/healthz"
# Expected: connection refused

# kubelet on workers starts logging errors:
#   "Unable to connect to the server: dial tcp 192.168.10.10:6443: connect: connection refused"
# Pods keep running (kubelet caches) but no new scheduling, no health reporting

# URGENT: Restore HAProxy immediately
ssh admin-lb "systemctl start haproxy"

# Verify recovery
kubectl get nodes
# All nodes should return to Ready within node-monitor-grace-period (default 40s)
```

**Step 4: Simulate network partition between workers and LB**
```bash
# Block worker -> LB traffic (simulates network partition)
ssh k8s-w1 "iptables -A OUTPUT -d 192.168.10.10 -j DROP"

# Worker loses all API server access
# kubelet stops reporting, node eventually marked NotReady
# Pods on this worker keep running but cannot be managed

# Restore
ssh k8s-w1 "iptables -D OUTPUT -d 192.168.10.10 -j DROP"
```

---

## Choosing the Right Configuration

| Criteria | Case 1 (Default) | Case 2 (External + Client) | Case 3 (External Only) |
|---|---|---|---|
| **External kubectl access** | Direct to CP node IP | Via HAProxy | Via HAProxy |
| **Internal resilience** | nginx-proxy per worker | nginx-proxy per worker | Depends on HAProxy |
| **SPOF risk** | None | HAProxy (external only) | HAProxy (everything) |
| **Infrastructure needed** | None | HAProxy host | HAProxy host + HA for HAProxy |
| **Failover speed** | 1s (nginx connect timeout) | 1s internal, ~3-5s external | ~3-5s (HAProxy health checks) |
| **Operational complexity** | Low | Medium | Medium-High |
| **Best for** | Dev/staging, simple prod | Production with external access | Large orgs with managed LB infra |

**Decision guide:**
- **Start with Case 1** for development, staging, or production clusters where all kubectl access is from within the network (SSH to a CP node, then kubectl).
- **Use Case 2** for production clusters that need a stable external API endpoint (CI/CD pipelines, developer workstations, monitoring systems) while keeping maximum internal resilience. This is the recommended production configuration.
- **Use Case 3** when you have a highly available load balancer infrastructure (cloud managed LB, keepalived pairs) and want a single, consistent API endpoint for all traffic. Avoid this unless your LB is itself highly available.

## Verifying HA Setup

### Check Leader Election
```bash
kubectl get lease -n kube-system
# Shows kube-controller-manager and kube-scheduler leases
# holderIdentity shows current leader
```

### Check etcd Cluster Health

Kubespray provides an `etcdctl.sh` wrapper script that automatically handles TLS certificate paths:

```bash
# Using the etcdctl.sh wrapper (recommended)
/usr/local/bin/etcdctl.sh endpoint health --cluster
# Expected: all endpoints show "is healthy"

/usr/local/bin/etcdctl.sh endpoint status --cluster --write-out=table
# Shows leader, DB size, raft term/index for each member
```

If the wrapper is unavailable, use etcdctl directly with full certificate paths:

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379 \
  endpoint health
```

### Check etcd Leader
```bash
/usr/local/bin/etcdctl.sh endpoint status --write-out=table
# IS LEADER column shows which member is the current leader

# Or manually:
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  --endpoints=https://192.168.10.11:2379 \
  endpoint status --write-out=table
```

### Check etcd Member List
```bash
/usr/local/bin/etcdctl.sh member list --write-out=table
# Verify all expected members are listed and started
```

## Testing Failover

### Control Plane Failover
```bash
# Stop kubelet on one control plane
ssh k8s-ctr1 "systemctl stop kubelet"

# Verify cluster still works
kubectl get nodes   # ctr1 shows NotReady
kubectl run test --image=nginx  # should succeed

# Restore
ssh k8s-ctr1 "systemctl start kubelet"
```

### etcd Failover
```bash
# Stop etcd on one node (WITH BACKUP FIRST)
ssh k8s-ctr1 "systemctl stop etcd"

# Check remaining cluster health
/usr/local/bin/etcdctl.sh endpoint health --cluster
# Or from ctr2:
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-k8s-ctr2.pem \
  --key=/etc/ssl/etcd/ssl/admin-k8s-ctr2-key.pem

# Verify writes work
kubectl create configmap test-ha --from-literal=key=value

# Restore
ssh k8s-ctr1 "systemctl start etcd"
```

## Stacked vs External etcd

| Aspect | Stacked | External |
|--------|---------|----------|
| Nodes required | 3 (min HA) | 6 (3 CP + 3 etcd) |
| Complexity | Lower | Higher |
| Resource isolation | Shared | Dedicated |
| Failure domain | Coupled | Independent |

**Recommendation:** Stacked for most cases. External when etcd performance is critical or you need independent scaling.

## Common Errors (Searchable)

```
etcdserver: request timed out
```
**Cause:** etcd quorum lost (too many nodes down). **Fix:** Restore failed nodes or restore from backup.

```
Unable to connect to the server: x509: certificate is valid for <IPs>, not <LB-IP>
```
**Cause:** External LB IP/domain not in API server certificate SANs. **Fix:** Add the LB address to `supplementary_addresses_in_ssl_keys` and re-run `ansible-playbook cluster.yml --tags "control-plane" --limit kube_control_plane`.

```
connection refused to <LB-IP>:6443
```
**Cause:** External LB is down or not configured. **Fix:** Check HAProxy/LB service status, verify backend health checks, ensure CP nodes are reachable from the LB.

```
Unable to connect to the server: dial tcp: lookup <hostname>: no such host
```
**Cause:** DNS not resolving LB hostname. **Fix:** Use IP address or fix DNS.

```
nginx-proxy: upstream timed out (110: Connection timed out)
```
**Cause:** All upstream CP nodes are unreachable from the worker. **Fix:** Check network connectivity between workers and CP nodes, verify API servers are running.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Even number of etcd nodes | No additional fault tolerance |
| Missing `supplementary_addresses_in_ssl_keys` for external LB | TLS errors (x509 certificate mismatch) when accessing via LB |
| Using Case 3 without HA for the LB itself | Single point of failure for entire cluster |
| Removing control plane without draining | Disrupted workloads |
| 2-node etcd cluster | Worse than 1 node (quorum=2, lose 1 = down) |
| Switching from Case 1 to Case 3 without full `cluster.yml` run | Workers still have stale nginx-proxy config |
| Not verifying certificate SANs after adding LB | Intermittent TLS failures that are hard to debug |
| Running `--tags "control-plane"` when switching to Case 3 | Incomplete reconfiguration -- need full `cluster.yml` |
