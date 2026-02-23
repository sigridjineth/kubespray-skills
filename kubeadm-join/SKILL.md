---
name: kubeadm-join
description: Use when joining worker or control-plane nodes to a Kubernetes cluster, troubleshooting TLS bootstrap, or debugging node join failures
---

# kubeadm join - Node Cluster Membership

## Overview

`kubeadm join` adds nodes to an existing cluster through bidirectional trust establishment.

**Core principle:** Join requires two-way trust - the node must verify the control plane (Discovery), and the control plane must verify the node (TLS Bootstrap).

## When to Use

- Adding worker nodes to cluster
- Adding control plane nodes for HA
- Troubleshooting node join failures
- Understanding TLS bootstrap mechanism

## Bidirectional Trust

```
┌─────────────────────────────────────────────────────────────┐
│                    BIDIRECTIONAL TRUST                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Discovery (Node → Control Plane)                          │
│  ─────────────────────────────────                         │
│  "Is this API Server legitimate?"                          │
│  Method: CA cert hash verification or kubeconfig file      │
│                                                             │
│  TLS Bootstrap (Control Plane → Node)                      │
│  ────────────────────────────────────                      │
│  "Is this node authorized to join?"                        │
│  Method: Bootstrap token → CSR → Certificate issuance      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Basic Join Command

From `kubeadm init` output:
```bash
kubeadm join 192.168.10.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxx...
```

| Component | Purpose |
|-----------|---------|
| `192.168.10.100:6443` | API Server endpoint |
| `--token` | Bootstrap token (used for both discovery and TLS bootstrap) |
| `--discovery-token-ca-cert-hash` | CA certificate fingerprint (prevents MITM) |

## Worker Node Join

### Phases
```
1. preflight     → Check requirements
2. discovery     → Get cluster-info, verify CA
3. TLS bootstrap → Authenticate with token, submit CSR, get certificate
4. kubelet-start → Configure and start kubelet
```

### Command
```bash
kubeadm join 192.168.10.100:6443 \
  --token 123456.1234567890123456 \
  --discovery-token-ca-cert-hash sha256:bd763182...
```

### Post-Join Files (Worker)
```
/etc/kubernetes/
├── kubelet.conf          # kubelet kubeconfig
└── pki/
    └── ca.crt            # Cluster CA (for verification)

/var/lib/kubelet/
├── config.yaml           # kubelet configuration
└── kubeadm-flags.env     # Extra kubelet flags
```

## Control Plane Node Join (HA)

### Additional Phases
```
1. preflight
2. discovery
3. control-plane-prepare  → Download certs, generate manifests
4. TLS bootstrap
5. etcd-join             → Add to etcd cluster
6. kubelet-start
7. control-plane-join    → Apply labels/taints
```

### Command
```bash
kubeadm join 192.168.10.100:6443 \
  --token 123456.1234567890123456 \
  --discovery-token-ca-cert-hash sha256:bd763182... \
  --control-plane \
  --certificate-key <certificate-key>
```

### Get Certificate Key
```bash
# On existing control plane
kubeadm init phase upload-certs --upload-certs
# Outputs: certificate-key: <64-char-hex>

# Or during initial kubeadm init
kubeadm init --upload-certs
```

## Token Management

### List Tokens
```bash
kubeadm token list
```

### Create New Token
```bash
# Simple
kubeadm token create

# With join command
kubeadm token create --print-join-command

# With custom TTL
kubeadm token create --ttl 2h
```

### Get CA Hash
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Discovery Methods

### Token-Based (Default)
```bash
kubeadm join ... --token xxx --discovery-token-ca-cert-hash sha256:xxx
```
- Fetches `cluster-info` ConfigMap from `kube-public` namespace
- Verifies CA with provided hash

### File-Based
```bash
kubeadm join ... --discovery-file /path/to/cluster-info.yaml
```
- Uses pre-distributed kubeconfig file
- For air-gapped or highly secure environments

### Unsafe (Not Recommended)
```bash
kubeadm join ... --discovery-token-unsafe-skip-ca-verification
```
- Skips CA verification
- Vulnerable to MITM attacks

## TLS Bootstrap Flow

```
New Node                           Control Plane
────────                           ─────────────
    │                                    │
    │──── 1. Get cluster-info ──────────▶│ (unauthenticated)
    │◀─── CA cert + API endpoint ────────│
    │                                    │
    │──── 2. Authenticate with token ───▶│
    │◀─── Temporary credentials ─────────│
    │                                    │
    │──── 3. Submit CSR ────────────────▶│
    │                                    │ (auto-approved)
    │◀─── 4. Signed certificate ─────────│
    │                                    │
    │──── 5. Use certificate ───────────▶│ (full kubelet access)
    │                                    │
```

## Configuration File Approach

```yaml
# join-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: "192.168.10.100:6443"
    token: "123456.1234567890123456"
    caCertHashes:
      - "sha256:bd763182..."
nodeRegistration:
  kubeletExtraArgs:
    - name: node-ip
      value: "192.168.10.101"
  criSocket: "unix:///run/containerd/containerd.sock"
```

```bash
kubeadm join --config=join-config.yaml
```

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Connection refused to 6443 | Firewall blocking | Open port 6443 on control plane |
| Token expired / "could not find JWS signature" | Default 24h TTL | `kubeadm token create --print-join-command` |
| CA hash mismatch | Wrong hash or different cluster | Get correct hash from control plane |
| TLS bootstrap timeout | Network/firewall issue | Check connectivity, firewalld |
| Node shows wrong IP | Multi-NIC, wrong default | Set `node-ip` in kubeletExtraArgs |
| x509 certificate error | Clock skew | Sync NTP on all nodes |

**Note:** The error "could not find a JWS signature in the cluster-info ConfigMap for token ID" means your bootstrap token has expired. The JWS signature is removed from cluster-info when the token expires.

## Troubleshooting Steps

### 1. Verify API Server Reachability
```bash
# From worker node
curl -k https://192.168.10.100:6443/healthz
# Expected: "ok"
```

### 2. Check Token Validity
```bash
# On control plane
kubeadm token list
# Check expiration time
```

### 3. Verify Network
```bash
# From worker
ping 192.168.10.100
ss -tlnp | grep 6443  # Should show kubelet after join
```

### 4. Check Firewall
```bash
# On control plane
systemctl is-active firewalld
# If active, open required ports or disable
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --reload
```

### 5. Check kubelet Logs
```bash
journalctl -u kubelet -f
```

## Reset and Retry

```bash
# On the node that failed to join
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet
iptables -F && iptables -t nat -F

# Re-attempt join
kubeadm join ...
```

## Verification Checklist

After successful join:
- [ ] Node appears in `kubectl get nodes`
- [ ] Node becomes Ready (after CNI installed)
- [ ] `kubelet` service running: `systemctl status kubelet`
- [ ] kube-proxy running: `crictl ps | grep proxy`
- [ ] `/etc/kubernetes/kubelet.conf` exists
- [ ] `/var/lib/kubelet/config.yaml` exists

## cluster-info ConfigMap

The bootstrap process depends on this publicly-accessible ConfigMap:

```bash
# View (works without authentication)
curl -k https://192.168.10.100:6443/api/v1/namespaces/kube-public/configmaps/cluster-info

# Contains:
# - kubeconfig: API Server address + CA certificate
# - jws-kubeconfig-<token>: Signed verification
```

This is the ONLY Kubernetes resource accessible without authentication.
