# Kubespray Installation

Production-ready Kubernetes installation for all tiers. All scripts are **idempotent** - safe to run multiple times.

> **Idempotency**: Kubespray playbooks are inherently idempotent. Running them again converges to the desired state. Helm uses `upgrade --install` and kubectl uses `apply` patterns.

## Version Matrix (January 2026)

| Component | Version | Notes |
|-----------|---------|-------|
| Kubespray | v2.29.1 | Latest stable |
| Kubernetes | v1.34.3 | Default for v2.29.1 |
| etcd | v3.5.26 | Bundled |
| containerd | v2.2.1 | Container runtime |
| Cilium | v1.18.6 | CNI + Gateway |
| Gateway API | v1.4.0 | GA release |
| cert-manager | v1.19.2 | TLS automation |
| Hetzner CCM | v1.22.0 | Optional: Hetzner provider |
| MetalLB | v0.14.9 | Optional: baremetal provider |

## Prerequisites

Run from **bastion server**:

```bash
# Install dependencies
apt update && apt install -y python3 python3-pip python3-venv git sshpass

# Clone Kubespray v2.29.1
git clone --branch v2.29.1 https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# Setup Python environment
python3 -m venv venv
source venv/bin/activate
pip install -U pip
pip install -r requirements.txt
```

## Production Ansible Configuration

Create optimized ansible.cfg for production deployments:

```bash
cat > ansible.cfg << 'EOF'
[defaults]
# Inventory
inventory = inventory/mycluster/hosts.yaml

# Performance
forks = 50
pipelining = True
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400

# SSH optimizations
host_key_checking = False
timeout = 30
remote_tmp = /tmp/.ansible-${USER}/tmp

# Output
stdout_callback = yaml
display_skipped_hosts = False
any_errors_fatal = True

# Logging
log_path = ./ansible.log

[ssh_connection]
# SSH multiplexing for speed
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ConnectionAttempts=100 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
pipelining = True
retries = 3

[persistent_connection]
connect_timeout = 30
command_timeout = 30
EOF
```

## Inventory Generation

### Option 1: Inventory Builder (Recommended)

```bash
# Declare node IPs
declare -a IPS=(10.0.1.1 10.0.1.2 10.0.1.3 10.0.2.1 10.0.2.2 10.0.2.3)

# Create inventory
CONFIG_FILE=inventory/mycluster/hosts.yaml \
  python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review generated inventory
cat inventory/mycluster/hosts.yaml
```

### Option 2: Manual Inventory

```yaml
# inventory/mycluster/hosts.yaml
all:
  hosts:
    k8s-master-1:
      ansible_host: 10.0.1.1
      ip: 10.0.1.1
      access_ip: 10.0.1.1
    k8s-master-2:
      ansible_host: 10.0.1.2
      ip: 10.0.1.2
      access_ip: 10.0.1.2
    k8s-master-3:
      ansible_host: 10.0.1.3
      ip: 10.0.1.3
      access_ip: 10.0.1.3
    k8s-worker-1:
      ansible_host: 10.0.2.1
      ip: 10.0.2.1
      access_ip: 10.0.2.1
    k8s-worker-2:
      ansible_host: 10.0.2.2
      ip: 10.0.2.2
      access_ip: 10.0.2.2
    k8s-worker-3:
      ansible_host: 10.0.2.3
      ip: 10.0.2.3
      access_ip: 10.0.2.3
  children:
    kube_control_plane:
      hosts:
        k8s-master-1:
        k8s-master-2:
        k8s-master-3:
    kube_node:
      hosts:
        k8s-worker-1:
        k8s-worker-2:
        k8s-worker-3:
    etcd:
      hosts:
        k8s-master-1:
        k8s-master-2:
        k8s-master-3:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

## Group Variables Configuration

### Kubernetes Cluster Settings

```bash
cat > inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml << 'EOF'
# Kubernetes version (Kubespray v2.29.1 default)
kube_version: v1.34.3

# Container runtime
container_manager: containerd

# Network settings
kube_pods_subnet: 10.233.64.0/18
kube_service_addresses: 10.233.0.0/18
kube_proxy_mode: ipvs

# Security
kubernetes_audit: true
podsecuritypolicy_enabled: false
kube_pod_security_use_default: true
kube_pod_security_default_enforce: baseline

# Addons
metrics_server_enabled: true
enable_nodelocaldns: true

# Feature gates
kube_feature_gates:
  - GracefulNodeShutdown=true
  - HPAContainerMetrics=true

# API server settings
kube_apiserver_enable_admission_plugins:
  - NodeRestriction
  - PodSecurity

# Kubelet settings
kubelet_max_pods: 110
kubelet_rotate_certificates: true
EOF
```

### Cilium CNI Configuration

```bash
cat > inventory/mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml << 'EOF'
# Cilium as CNI
kube_network_plugin: cilium
cilium_version: v1.18.6

# Hubble observability
cilium_enable_hubble: true
cilium_hubble_enabled: true
cilium_hubble_relay_enabled: true
cilium_hubble_ui_enabled: true

# Gateway API support
cilium_gateway_api_enabled: true
cilium_enable_gateway_api_support: true

# Performance tuning (production)
cilium_enable_bpf_masquerade: true
cilium_kube_proxy_replacement: strict
cilium_tunnel_mode: disabled
cilium_auto_direct_node_routes: true

# Native routing (better performance on Hetzner)
cilium_native_routing_cidr: 10.0.0.0/16

# Health checking
cilium_enable_endpoint_health_checking: true
cilium_enable_health_checking: true

# Bandwidth manager (QoS)
cilium_enable_bandwidth_manager: true

# IP masquerade
cilium_enable_ipv4_masquerade: true
EOF
```

### etcd Configuration

```bash
cat > inventory/mycluster/group_vars/etcd.yml << 'EOF'
# etcd deployment
etcd_deployment_type: host
etcd_data_dir: /var/lib/etcd

# Compaction (prevent unbounded growth)
etcd_compaction_retention: "8"
etcd_quota_backend_bytes: "8589934592"

# Snapshots
etcd_snapshot_count: "10000"

# Events cluster (production: separate etcd for events)
etcd_events_cluster_enabled: false
EOF
```

### All Nodes Configuration

```bash
cat > inventory/mycluster/group_vars/all/all.yml << 'EOF'
# Ansible SSH settings
ansible_user: root
ansible_ssh_private_key_file: ~/.ssh/id_ed25519

# Timezone
ntp_enabled: true
ntp_timezone: UTC

# Download settings (speed up installation)
download_run_once: true
download_localhost: false
download_always_pull: false
download_force_cache: false

# Container registry mirrors (optional, for air-gapped)
# containerd_registries_mirrors:
#   - prefix: docker.io
#     mirrors:
#       - host: https://registry.example.com
#         capabilities: ["pull", "resolve"]
EOF
```

## Unified Installation Script

```bash
#!/bin/bash
# scripts/install-cluster.sh
# Run from: bastion server

set -euo pipefail

PROJECT="k8s"
TIER="small"
KUBESPRAY_VERSION="v2.29.1"

while [[ $# -gt 0 ]]; do
  case $1 in
    --project) PROJECT="$2"; shift 2 ;;
    --tier) TIER="$2"; shift 2 ;;
    *) shift ;;
  esac
done

echo "============================================"
echo "Kubernetes Cluster Installation"
echo "============================================"
echo "Project: $PROJECT"
echo "Tier: $TIER"
echo "Kubespray: $KUBESPRAY_VERSION"
echo "============================================"

# Determine node configuration based on tier
case "$TIER" in
  minimal)
    MASTERS=1
    WORKERS=1
    ETCD_NODES=1
    MASTER_SCHEDULABLE=true
    ;;
  small)
    MASTERS=1
    WORKERS=2
    ETCD_NODES=1
    MASTER_SCHEDULABLE=false
    ;;
  medium)
    MASTERS=3
    WORKERS=2
    ETCD_NODES=3
    MASTER_SCHEDULABLE=false
    ;;
  production)
    MASTERS=3
    WORKERS=3
    ETCD_NODES=3
    MASTER_SCHEDULABLE=false
    ;;
  *)
    echo "Unknown tier: $TIER"
    echo "Use: minimal | small | medium | production"
    exit 1
    ;;
esac

echo "Masters: $MASTERS, Workers: $WORKERS, etcd: $ETCD_NODES"
echo ""

# Clone or update Kubespray
if [[ ! -d kubespray ]]; then
  git clone --branch ${KUBESPRAY_VERSION} https://github.com/kubernetes-sigs/kubespray.git
fi
cd kubespray

# Ensure correct version
git fetch --tags
git checkout ${KUBESPRAY_VERSION}

# Setup Python venv
if [[ ! -d venv ]]; then
  python3 -m venv venv
fi
source venv/bin/activate
pip install -U pip -q
pip install -r requirements.txt -q

# Create production ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory/mycluster/hosts.yaml
forks = 50
pipelining = True
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400
host_key_checking = False
timeout = 30
stdout_callback = yaml
display_skipped_hosts = False
any_errors_fatal = True
log_path = ./ansible.log

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ConnectionAttempts=100 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
pipelining = True
retries = 3
EOF

# Create inventory
cp -r inventory/sample inventory/mycluster 2>/dev/null || true

# Generate hosts.yaml
cat > inventory/mycluster/hosts.yaml << EOF
all:
  hosts:
EOF

# Add master nodes
for i in $(seq 1 $MASTERS); do
  cat >> inventory/mycluster/hosts.yaml << EOF
    ${PROJECT}-master-${i}:
      ansible_host: 10.0.1.${i}
      ip: 10.0.1.${i}
      access_ip: 10.0.1.${i}
EOF
done

# Add worker nodes
for i in $(seq 1 $WORKERS); do
  cat >> inventory/mycluster/hosts.yaml << EOF
    ${PROJECT}-worker-${i}:
      ansible_host: 10.0.2.${i}
      ip: 10.0.2.${i}
      access_ip: 10.0.2.${i}
EOF
done

# Add group definitions
cat >> inventory/mycluster/hosts.yaml << EOF
  children:
    kube_control_plane:
      hosts:
EOF

for i in $(seq 1 $MASTERS); do
  echo "        ${PROJECT}-master-${i}:" >> inventory/mycluster/hosts.yaml
done

cat >> inventory/mycluster/hosts.yaml << EOF
    kube_node:
      hosts:
EOF

if [[ "$MASTER_SCHEDULABLE" == "true" ]]; then
  for i in $(seq 1 $MASTERS); do
    echo "        ${PROJECT}-master-${i}:" >> inventory/mycluster/hosts.yaml
  done
fi

for i in $(seq 1 $WORKERS); do
  echo "        ${PROJECT}-worker-${i}:" >> inventory/mycluster/hosts.yaml
done

cat >> inventory/mycluster/hosts.yaml << EOF
    etcd:
      hosts:
EOF

for i in $(seq 1 $ETCD_NODES); do
  echo "        ${PROJECT}-master-${i}:" >> inventory/mycluster/hosts.yaml
done

cat >> inventory/mycluster/hosts.yaml << EOF
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
EOF

echo "Generated inventory:"
cat inventory/mycluster/hosts.yaml
echo ""

# Configure Cilium CNI
cat > inventory/mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml << 'EOF'
kube_network_plugin: cilium
cilium_version: v1.18.6
cilium_enable_hubble: true
cilium_hubble_enabled: true
cilium_hubble_relay_enabled: true
cilium_hubble_ui_enabled: true
cilium_gateway_api_enabled: true
cilium_enable_gateway_api_support: true
cilium_enable_bpf_masquerade: true
cilium_kube_proxy_replacement: strict
cilium_tunnel_mode: disabled
cilium_auto_direct_node_routes: true
cilium_native_routing_cidr: 10.0.0.0/16
cilium_enable_endpoint_health_checking: true
cilium_enable_health_checking: true
cilium_enable_bandwidth_manager: true
cilium_enable_ipv4_masquerade: true
EOF

# Kubernetes configuration
cat > inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml << 'EOF'
kube_version: v1.34.3
container_manager: containerd
kube_pods_subnet: 10.233.64.0/18
kube_service_addresses: 10.233.0.0/18
kube_proxy_mode: ipvs
kubernetes_audit: true
metrics_server_enabled: true
enable_nodelocaldns: true
podsecuritypolicy_enabled: false
kube_pod_security_use_default: true
kube_pod_security_default_enforce: baseline
kubelet_max_pods: 110
kubelet_rotate_certificates: true
kube_feature_gates:
  - GracefulNodeShutdown=true
  - HPAContainerMetrics=true
kube_apiserver_enable_admission_plugins:
  - NodeRestriction
  - PodSecurity
EOF

# etcd configuration
cat > inventory/mycluster/group_vars/etcd.yml << 'EOF'
etcd_deployment_type: host
etcd_data_dir: /var/lib/etcd
etcd_compaction_retention: "8"
etcd_quota_backend_bytes: "8589934592"
etcd_snapshot_count: "10000"
etcd_events_cluster_enabled: false
EOF

# All nodes configuration
cat > inventory/mycluster/group_vars/all/all.yml << 'EOF'
ansible_user: root
ntp_enabled: true
ntp_timezone: UTC
download_run_once: true
download_localhost: false
EOF

# Create logs directory
mkdir -p ../logs

echo ""
echo "=== Running Kubespray Playbook ==="

# Run the playbook
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  cluster.yml 2>&1 | tee ../logs/kubespray-install.log

RESULT=$?

if [[ $RESULT -ne 0 ]]; then
  echo ""
  echo "ERROR: Kubespray installation failed!"
  echo "Check logs: ../logs/kubespray-install.log"
  exit 1
fi

echo ""
echo "=== Kubernetes Cluster Installed ==="

# Copy kubeconfig to bastion
mkdir -p ~/.kube
scp ${PROJECT}-master-1:/etc/kubernetes/admin.conf ~/.kube/config 2>/dev/null || \
  ssh ${PROJECT}-master-1 "cat /etc/kubernetes/admin.conf" > ~/.kube/config
chmod 600 ~/.kube/config

# Wait for nodes
echo "Waiting for nodes to be ready..."
kubectl wait --for=condition=Ready nodes --all --timeout=300s

# Verify cluster
echo ""
echo "=== Cluster Status ==="
kubectl get nodes -o wide
echo ""
kubectl get pods -A

echo ""
echo "=== Cilium Status ==="
kubectl -n kube-system wait --for=condition=Ready pod -l k8s-app=cilium --timeout=120s
cilium status --wait 2>/dev/null || kubectl get pods -n kube-system -l k8s-app=cilium

echo ""
echo "============================================"
echo "Kubernetes cluster ready!"
echo "kubectl commands work from bastion"
echo "============================================"
```

## Post-Installation

### Install Gateway API CRDs

```bash
#!/bin/bash
# scripts/install-gateway-api.sh
# Run from: bastion server

set -euo pipefail

GATEWAY_API_VERSION="v1.4.0"

echo "=== Installing Gateway API ${GATEWAY_API_VERSION} ==="

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/standard-install.yaml

kubectl wait --for=condition=Established crd/gatewayclasses.gateway.networking.k8s.io --timeout=60s
kubectl wait --for=condition=Established crd/gateways.gateway.networking.k8s.io --timeout=60s
kubectl wait --for=condition=Established crd/httproutes.gateway.networking.k8s.io --timeout=60s

cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
EOF

echo "Gateway API installed!"
kubectl get gatewayclass
```

### Install LoadBalancer (Multi-Provider)

```bash
#!/bin/bash
# scripts/install-loadbalancer.sh
# Run from: bastion server
# Idempotent: safe to run multiple times

set -euo pipefail

PROVIDER="hetzner"
METALLB_VERSION="v0.14.9"
IP_RANGE=""
NETWORK_NAME="k8s-network"

usage() {
  cat << EOF
Usage: $0 --provider <provider> [options]

Providers:
  hetzner     Hetzner Cloud Controller Manager (default)
  aws         AWS Cloud Provider
  gcp         GCP Cloud Provider
  azure       Azure Cloud Provider
  baremetal   MetalLB for bare metal / other clouds

Options:
  --provider <name>     Cloud provider (required)
  --ip-range <range>    IP range for MetalLB (required for baremetal)
                        Format: 192.168.1.240-192.168.1.250 or 192.168.1.0/28
  --network <name>      Network name (Hetzner only, default: k8s-network)

Environment variables:
  HCLOUD_TOKEN          Hetzner API token (required for hetzner)
  AWS_ACCESS_KEY_ID     AWS credentials (required for aws)
  AWS_SECRET_ACCESS_KEY
  GOOGLE_APPLICATION_CREDENTIALS  GCP service account (required for gcp)
  AZURE_*               Azure credentials (required for azure)

Examples:
  $0 --provider hetzner
  $0 --provider aws
  $0 --provider baremetal --ip-range 192.168.1.240-192.168.1.250
EOF
  exit 1
}

while [[ $# -gt 0 ]]; do
  case $1 in
    --provider) PROVIDER="$2"; shift 2 ;;
    --ip-range) IP_RANGE="$2"; shift 2 ;;
    --network) NETWORK_NAME="$2"; shift 2 ;;
    -h|--help) usage ;;
    *) echo "Unknown option: $1"; usage ;;
  esac
done

echo "============================================"
echo "LoadBalancer Installation"
echo "============================================"
echo "Provider: $PROVIDER"
echo "============================================"

install_hetzner() {
  echo "=== Installing Hetzner Cloud Controller Manager ==="

  [[ -z "${HCLOUD_TOKEN:-}" ]] && { echo "ERROR: HCLOUD_TOKEN required"; exit 1; }

  # Create/update secret (idempotent)
  kubectl -n kube-system create secret generic hcloud \
    --from-literal=token="$HCLOUD_TOKEN" \
    --from-literal=network="$NETWORK_NAME" \
    --dry-run=client -o yaml | kubectl apply -f -

  # Deploy CCM (idempotent - kubectl apply)
  kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/download/v1.22.0/ccm-networks.yaml

  kubectl -n kube-system wait --for=condition=Ready pod -l app=hcloud-cloud-controller-manager --timeout=120s

  echo "Hetzner CCM installed!"
  kubectl get pods -n kube-system -l app=hcloud-cloud-controller-manager
}

install_aws() {
  echo "=== Installing AWS Cloud Provider ==="

  # AWS Cloud Provider is typically configured via Kubespray
  # This installs the AWS Load Balancer Controller for advanced LB features

  helm repo add eks https://aws.github.io/eks-charts 2>/dev/null || true
  helm repo update

  # Install AWS Load Balancer Controller (idempotent)
  helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace kube-system \
    --set clusterName="${CLUSTER_NAME:-k8s}" \
    --set serviceAccount.create=true \
    --wait

  echo "AWS Load Balancer Controller installed!"
  kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
}

install_gcp() {
  echo "=== Installing GCP Cloud Provider ==="

  # GCP Cloud Provider is typically configured via Kubespray
  # Verify it's working

  echo "GCP Cloud Provider should be configured during cluster setup."
  echo "Verifying cloud-controller-manager..."

  kubectl get pods -n kube-system -l component=cloud-controller-manager || \
    echo "Note: GCP CCM may be running as part of the control plane"

  echo "GCP Cloud Provider ready!"
}

install_azure() {
  echo "=== Installing Azure Cloud Provider ==="

  # Azure Cloud Provider is typically configured via Kubespray
  # Verify it's working

  echo "Azure Cloud Provider should be configured during cluster setup."
  echo "Verifying cloud-controller-manager..."

  kubectl get pods -n kube-system -l component=cloud-controller-manager || \
    echo "Note: Azure CCM may be running as part of the control plane"

  echo "Azure Cloud Provider ready!"
}

install_metallb() {
  echo "=== Installing MetalLB ${METALLB_VERSION} ==="

  [[ -z "$IP_RANGE" ]] && { echo "ERROR: --ip-range required for baremetal"; exit 1; }

  # Install MetalLB via Helm (idempotent)
  helm repo add metallb https://metallb.github.io/metallb 2>/dev/null || true
  helm repo update

  helm upgrade --install metallb metallb/metallb \
    --namespace metallb-system \
    --create-namespace \
    --version ${METALLB_VERSION} \
    --wait

  # Wait for controller and speaker pods
  kubectl -n metallb-system wait --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=120s
  kubectl -n metallb-system wait --for=condition=Ready pod -l app.kubernetes.io/component=speaker --timeout=120s

  # Configure IP address pool (idempotent - kubectl apply)
  cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - ${IP_RANGE}
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF

  echo "MetalLB installed with IP range: $IP_RANGE"
  kubectl get pods -n metallb-system
  kubectl get ipaddresspool -n metallb-system
}

# Execute based on provider
case "$PROVIDER" in
  hetzner) install_hetzner ;;
  aws) install_aws ;;
  gcp) install_gcp ;;
  azure) install_azure ;;
  baremetal|metallb) install_metallb ;;
  *)
    echo "ERROR: Unknown provider: $PROVIDER"
    echo "Use: hetzner | aws | gcp | azure | baremetal"
    exit 1
    ;;
esac

echo ""
echo "============================================"
echo "LoadBalancer setup complete!"
echo "Provider: $PROVIDER"
echo "============================================"

# Verify LoadBalancer works
echo ""
echo "To test, create a LoadBalancer service:"
echo "  kubectl create deployment nginx --image=nginx"
echo "  kubectl expose deployment nginx --port=80 --type=LoadBalancer"
echo "  kubectl get svc nginx -w"
```

### Install cert-manager

```bash
#!/bin/bash
# scripts/install-cert-manager.sh
# Run from: bastion server

set -euo pipefail

CERT_MANAGER_VERSION="v1.19.2"

echo "=== Installing cert-manager ${CERT_MANAGER_VERSION} ==="

helm repo add jetstack https://charts.jetstack.io 2>/dev/null || true
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version ${CERT_MANAGER_VERSION} \
  --set crds.enabled=true \
  --set dns01RecursiveNameservers="1.1.1.1:53,8.8.8.8:53" \
  --set dns01RecursiveNameserversOnly=true \
  --wait

echo "cert-manager installed!"
kubectl get pods -n cert-manager
```

## Verify Installation

```bash
# All commands run from bastion server

# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -A

# Check Cilium
cilium status
cilium connectivity test

# Check Gateway API
kubectl get gatewayclass
kubectl get gateway -A

# Check etcd health
kubectl -n kube-system exec -it etcd-k8s-master-1 -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

## Cluster Access

All cluster access is through the **bastion server**:

```bash
# SSH to bastion
ssh user@bastion-ip

# kubectl works directly from bastion
kubectl get nodes
kubectl get pods -A

# Access via VPN (Headscale)
tailscale up --login-server https://vpn.example.com --authkey <KEY>
kubectl get nodes
```

## Production Checklist

- [ ] 3+ control plane nodes for HA
- [ ] etcd on dedicated nodes or co-located with control plane
- [ ] Anti-affinity rules for critical workloads
- [ ] NetworkPolicies enabled (Cilium default)
- [ ] Pod Security Standards enforced (baseline)
- [ ] Audit logging enabled
- [ ] Node certificates rotation enabled
- [ ] LoadBalancer configured (CCM or MetalLB)
- [ ] cert-manager for TLS automation
- [ ] Gateway API for ingress
- [ ] Backup strategy for etcd
