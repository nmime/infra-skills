# DNS Configuration

For DNS management, see **hetzner-infra** skill.

This file covers VPN-specific DNS (Headscale Magic DNS).

## Headscale Private DNS

VPN clients get private DNS resolution via Headscale:

```yaml
# /etc/headscale/config.yaml
dns:
  magic_dns: true
  base_domain: internal
  
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
  
  # Private records - VPN clients only
  extra_records:
    - name: gitlab.example.com
      type: A
      value: 10.0.10.1
    - name: argocd.example.com
      type: A
      value: 10.0.10.2
    - name: grafana.example.com
      type: A
      value: 10.0.10.3
    - name: vault.example.com
      type: A
      value: 10.0.10.4
    - name: minio.example.com
      type: A
      value: 10.0.10.5
```

## How It Works

```
Without VPN:
  gitlab.example.com → Public IP (blocked by firewall)

With VPN:
  gitlab.example.com → 10.0.10.1 (via Headscale DNS)
```