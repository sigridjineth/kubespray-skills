---
name: kubespray-certificates
description: Use when Kubernetes certificates are expiring, checking certificate expiration dates, setting up auto-renewal, or manually renewing certificates. Use when seeing x509 certificate errors.
---

# Kubespray Certificate Management

## Overview

Kubernetes uses TLS certificates for all internal communication. Kubespray-deployed clusters have certificates that expire after 1 year by default. Expired certificates cause immediate cluster failure.

**Core principle:** Enable auto-renewal during deployment. If certificates expire, manual renewal is required and time-sensitive.

## When to Use

- Checking certificate expiration dates
- Enabling certificate auto-renewal
- Manually renewing expired certificates
- Troubleshooting x509 certificate errors

**Not for:** Initial deployment (use kubespray-deployment), cluster upgrades (use kubespray-operations)

## Certificate Types

| Certificate | Location | Purpose |
|-------------|----------|---------|
| CA cert | `/etc/kubernetes/pki/ca.crt` | Signs all other certs |
| API server | `/etc/kubernetes/pki/apiserver.crt` | API server identity |
| kubelet client | `/etc/kubernetes/pki/apiserver-kubelet-client.crt` | API → kubelet auth |
| etcd CA | `/etc/ssl/etcd/ssl/ca.pem` | Signs etcd certs |
| etcd member | `/etc/ssl/etcd/ssl/member-*.pem` | etcd peer auth |

## Checking Expiration

### Using kubeadm

```bash
kubeadm certs check-expiration
```

Output shows all certificates with expiration dates.

### Using openssl

```bash
# Check specific certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Check etcd certificate
openssl x509 -in /etc/ssl/etcd/ssl/member-$(hostname).pem -noout -dates

# One-liner for all K8s PKI certs
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates
done
```

### Check Certificate Details

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity
```

## Enabling Auto-Renewal (Recommended)

### During Initial Deployment

Add to `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
auto_renew_certificates: true
auto_renew_certificates_systemd_calendar: "Mon *-*-1,15 03:00:00"
```

This creates a systemd timer that runs twice monthly.

### Verify Timer is Active

```bash
systemctl list-timers | grep cert
# Shows k8s-cert-renew.timer

systemctl status k8s-cert-renew.timer
```

### What Auto-Renewal Does

The timer runs:
```bash
kubeadm certs renew all
```

Then restarts control plane components via kubelet.

## Manual Certificate Renewal

### When Auto-Renewal is Not Configured

Run Kubespray's certificate renewal playbook:

```bash
cd kubespray
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --tags=k8s-secrets,kubernetes/control-plane \
  -b
```

### Using kubeadm Directly

On each control plane node:

```bash
# Renew all certificates
kubeadm certs renew all

# Restart control plane (method depends on deployment)
# For static pods:
mv /etc/kubernetes/manifests/*.yaml /tmp/
sleep 10
mv /tmp/*.yaml /etc/kubernetes/manifests/

# Verify
kubectl get nodes
```

### Renewing Specific Certificates

```bash
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client
kubeadm certs renew front-proxy-client
kubeadm certs renew admin.conf
```

## etcd Certificate Renewal

etcd certificates are separate from Kubernetes certificates.

### Check etcd Certificate Expiration

```bash
openssl x509 -in /etc/ssl/etcd/ssl/member-$(hostname).pem -noout -dates
openssl x509 -in /etc/ssl/etcd/ssl/admin-$(hostname).pem -noout -dates
```

### Renew etcd Certificates via Kubespray

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --tags=etcd \
  -b
```

## Troubleshooting Certificate Issues

### Error: x509 certificate has expired

```
x509: certificate has expired or is not yet valid
```

**Immediate fix:**
```bash
# On control plane
kubeadm certs renew all

# Restart kubelet
systemctl restart kubelet

# Wait for static pods
sleep 30
kubectl get nodes
```

### Error: certificate signed by unknown authority

```
x509: certificate signed by unknown authority
```

Indicates CA mismatch - fundamental trust chain issue. Options:
1. **Full reset** - Use `reset.yml` and redeploy (loses all data)
2. **CA rotation** - Advanced procedure, see [Kubernetes CA rotation docs](https://kubernetes.io/docs/tasks/tls/manual-rotation-of-ca-certificates/)

**Warning:** CA rotation is complex and can break the cluster if done incorrectly. Test in non-production first.

### kubectl: Unable to connect to server

If certificates expired and kubectl fails:

```bash
# Renew admin.conf certificate
kubeadm certs renew admin.conf

# Copy new kubeconfig
cp /etc/kubernetes/admin.conf ~/.kube/config
```

## Certificate Validity Period

Kubespray default: **1 year** for most certificates, **10 years** for CA.

To change (before deployment):
```yaml
# group_vars/k8s_cluster/k8s-cluster.yml
kube_cert_validity_duration: 365  # days
```

## Systemd Timer Configuration

```ini
# /etc/systemd/system/k8s-cert-renew.timer
[Unit]
Description=Kubernetes Certificate Renewal

[Timer]
OnCalendar=Mon *-*-1,15 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/k8s-cert-renew.service
[Unit]
Description=Renew Kubernetes certificates

[Service]
Type=oneshot
ExecStart=/usr/local/bin/kube-scripts/k8s-certs-renew.sh
```

## Monitoring Certificate Expiration

### Prometheus Alert Rule

```yaml
groups:
- name: kubernetes-certificates
  rules:
  - alert: KubernetesCertificateExpiringSoon
    expr: apiserver_client_certificate_expiration_seconds_bucket{le="604800"} > 0
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes certificate expiring within 7 days"
```

### Simple Monitoring Script

```bash
#!/bin/bash
# cert-check.sh - Run via cron, alert if < 30 days remaining

for cert in /etc/kubernetes/pki/*.crt; do
  expiry=$(openssl x509 -in "$cert" -noout -enddate | cut -d= -f2)
  expiry_epoch=$(date -d "$expiry" +%s)
  now_epoch=$(date +%s)
  days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

  if [ $days_left -lt 30 ]; then
    echo "WARNING: $cert expires in $days_left days"
  fi
done
```

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Not enabling `auto_renew_certificates` | Certs expire, cluster down |
| Forgetting etcd certs | etcd fails, cluster unrecoverable |
| Not restarting components after renewal | Old certs still in use |
| Renewing only some certs | Mismatched certs, auth failures |
| Ignoring expiration warnings | Scrambling at midnight when cluster dies |

## Quick Reference

```bash
# Check all expirations
kubeadm certs check-expiration

# Renew all certs
kubeadm certs renew all

# Enable auto-renewal in Kubespray
auto_renew_certificates: true

# Verify timer
systemctl list-timers | grep cert
```
