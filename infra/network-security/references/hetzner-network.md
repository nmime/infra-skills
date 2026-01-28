# Hetzner Private Network Setup

## Create Network

```bash
#!/bin/bash
# scripts/setup-hetzner-network.sh

set -euo pipefail

NETWORK_NAME="k8s-private"
NETWORK_CIDR="10.0.0.0/8"
SUBNET_CIDR="10.0.0.0/16"
NETWORK_ZONE="eu-central"  # Covers fsn1, nbg1, hel1

echo "=== Creating Hetzner Private Network ==="

# Create network
hcloud network create \
  --name ${NETWORK_NAME} \
  --ip-range ${NETWORK_CIDR}

# Create subnet
hcloud network add-subnet ${NETWORK_NAME} \
  --type cloud \
  --network-zone ${NETWORK_ZONE} \
  --ip-range ${SUBNET_CIDR}

echo "=== Network Created ==="
hcloud network describe ${NETWORK_NAME}
```

## Network Layout

```
Hetzner Private Network: 10.0.0.0/8
└── Subnet: 10.0.0.0/16
    │
    ├── 10.0.0.0/24   - Infrastructure
    │   ├── 10.0.0.1  - Gateway (Hetzner)
    │   ├── 10.0.0.2  - Bastion
    │   └── 10.0.0.3  - (reserved)
    │
    ├── 10.0.1.0/24   - K8s Control Plane
    │   ├── 10.0.1.1  - master-1
    │   ├── 10.0.1.2  - master-2
    │   └── 10.0.1.3  - master-3
    │
    ├── 10.0.2.0/24   - K8s Workers
    │   ├── 10.0.2.1  - worker-1
    │   ├── 10.0.2.2  - worker-2
    │   └── 10.0.2.3  - worker-3
    │
    └── 10.0.10.0/24  - K8s Services (MetalLB)
        ├── 10.0.10.1 - GitLab
        ├── 10.0.10.2 - ArgoCD
        ├── 10.0.10.3 - Grafana
        └── ...
```

## Attach Servers to Network

```bash
#!/bin/bash
# scripts/attach-servers-to-network.sh

NETWORK="k8s-private"

# Bastion (with specific IP)
hcloud server attach-to-network bastion \
  --network ${NETWORK} \
  --ip 10.0.0.2

# Masters
hcloud server attach-to-network master-1 --network ${NETWORK} --ip 10.0.1.1
hcloud server attach-to-network master-2 --network ${NETWORK} --ip 10.0.1.2
hcloud server attach-to-network master-3 --network ${NETWORK} --ip 10.0.1.3

# Workers
hcloud server attach-to-network worker-1 --network ${NETWORK} --ip 10.0.2.1
hcloud server attach-to-network worker-2 --network ${NETWORK} --ip 10.0.2.2
hcloud server attach-to-network worker-3 --network ${NETWORK} --ip 10.0.2.3
```

## Route Configuration

Hetzner automatically routes traffic between servers in the same network.

```bash
# On each server, private interface is ens10
# Verify:
ip addr show ens10

# Should show:
# inet 10.0.x.x/32 scope global ens10
```