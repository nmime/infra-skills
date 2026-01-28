# Network Architecture

## Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HETZNER CLOUD                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   INTERNET                                                                  │
│       │                                                                     │
│       │                                                                     │
│   ┌───┴────────────────────────────────────────────────────┐                │
│   │                                                        │                │
│   │   ┌─────────────────┐      ┌─────────────────────────┐ │                │
│   │   │    BASTION      │      │    LOAD BALANCER        │ │                │
│   │   │                 │      │                         │ │                │
│   │   │ vpn.example.com │      │ *.example.com (apps)    │ │                │
│   │   │                 │      │                         │ │                │
│   │   │ Services:       │      │ Target: K8s Gateway     │ │                │
│   │   │ • Headscale VPN │      │ (10.0.10.100:443)       │ │                │
│   │   │                 │      │                         │ │                │
│   │   │ Public IP: Yes  │      │ Public IP: Yes          │ │                │
│   │   │ Private: 10.0.0.2      │ Private: attached       │ │                │
│   │   └────────┬────────┘      └────────────┬────────────┘ │                │
│   │            │                            │              │                │
│   └────────────┼────────────────────────────┼──────────────┘                │
│                │                            │                               │
│                │    PRIVATE NETWORK         │                               │
│                │    (10.0.0.0/16)           │                               │
│                │                            │                               │
│   ┌────────────┴────────────────────────────┴──────────────────────────┐    │
│   │                                                                    │    │
│   │   KUBERNETES CLUSTER (No Public IPs)                               │    │
│   │                                                                    │    │
│   │   Control Plane: 10.0.1.1, 10.0.1.2, 10.0.1.3                     │    │
│   │   Workers: 10.0.2.1, 10.0.2.2, 10.0.2.3                           │    │
│   │                                                                    │    │
│   │   ┌────────────────────────────────────────────────────────────┐  │    │
│   │   │  CILIUM GATEWAY (10.0.10.100)                              │  │    │
│   │   │                                                            │  │    │
│   │   │  Routes all HTTP/HTTPS traffic to services                 │  │    │
│   │   └────────────────────────────────────────────────────────────┘  │    │
│   │                                                                    │    │
│   │   ┌──────────────────────┐  ┌──────────────────────┐              │    │
│   │   │  PUBLIC SERVICES     │  │  PRIVATE SERVICES    │              │    │
│   │   │  (via Load Balancer) │  │  (via VPN only)      │              │    │
│   │   │                      │  │                      │              │    │
│   │   │  • Frontend App      │  │  • GitLab            │              │    │
│   │   │  • Backend API       │  │  • ArgoCD            │              │    │
│   │   │  • MinIO S3 API      │  │  • Grafana           │              │    │
│   │   │  • Container Registry│  │  • Vault             │              │    │
│   │   │                      │  │  • MinIO Console     │              │    │
│   │   │                      │  │  • PostgreSQL        │              │    │
│   │   └──────────────────────┘  └──────────────────────┘              │    │
│   │                                                                    │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Network Segments

| Segment | CIDR | Purpose |
|---------|------|--------|
| Infrastructure | 10.0.0.0/24 | Bastion, DNS |
| Control Plane | 10.0.1.0/24 | K8s masters |
| Workers | 10.0.2.0/24 | K8s workers |
| Services | 10.0.10.0/24 | MetalLB pool |
| Pods | 10.233.64.0/18 | Pod CIDR |
| Services | 10.233.0.0/18 | Service CIDR |

## Public Endpoints

| Endpoint | Points To | Purpose |
|----------|-----------|--------|
| Load Balancer IP | Cilium Gateway | Public apps |
| Bastion IP | Headscale | VPN access |

## Internal Endpoints

| Service | IP | Port |
|---------|-----|------|
| Cilium Gateway | 10.0.10.100 | 80, 443 |
| GitLab | 10.0.10.1 | 443 |
| ArgoCD | 10.0.10.2 | 443 |
| Grafana | 10.0.10.3 | 443 |
| MinIO | 10.0.10.4 | 9000, 9001 |
| Vault | 10.0.10.5 | 8200 |
| PostgreSQL | 10.0.10.6 | 5432 |