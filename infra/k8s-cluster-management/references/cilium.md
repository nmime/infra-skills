# Cilium CNI Setup

Advanced eBPF-based networking with Cilium.

Run all commands from **bastion server**.

## Version Information (January 2026)

| Component | Version |
|-----------|---------|
| Cilium | v1.18.6 |
| Cilium CLI | v0.18.4 |
| Hubble CLI | v1.18.6 |
| Gateway API | v1.4.0 |

## Install Cilium CLI

```bash
#!/bin/bash
# scripts/install-cilium-cli.sh
# Run from: bastion server

CILIUM_CLI_VERSION="v0.18.4"
HUBBLE_VERSION="v1.18.6"

CLI_ARCH=amd64

# Cilium CLI
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Hubble CLI
curl -L --fail --remote-name-all \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

echo "Installed: cilium $(cilium version --client), hubble $(hubble version)"
```

## Standalone Installation

Cilium is installed via Kubespray. For manual installation:

```bash
#!/bin/bash
# scripts/install-cilium.sh
# Run from: bastion server

set -euo pipefail

CILIUM_VERSION="1.18.6"
GATEWAY_API_VERSION="v1.4.0"
API_SERVER_IP="${1:-10.0.1.1}"
API_SERVER_PORT="${2:-6443}"

echo "=== Installing Gateway API CRDs ${GATEWAY_API_VERSION} ==="
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/standard-install.yaml
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/experimental-install.yaml

kubectl wait --for condition=established --timeout=60s \
  crd/gateways.gateway.networking.k8s.io \
  crd/httproutes.gateway.networking.k8s.io

echo "=== Installing Cilium ${CILIUM_VERSION} ==="
helm repo add cilium https://helm.cilium.io/
helm repo update

helm upgrade --install cilium cilium/cilium \
  --version ${CILIUM_VERSION} \
  --namespace kube-system \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true \
  --set gatewayAPI.secretsNamespace.create=true \
  --set gatewayAPI.secretsNamespace.name=cilium-secrets \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set ipam.mode=cluster-pool \
  --set tunnel=disabled \
  --set autoDirectNodeRoutes=true \
  --set bpf.masquerade=true \
  --set loadBalancer.mode=dsr \
  --set loadBalancer.algorithm=maglev \
  --set bandwidthManager.enabled=true \
  --set operator.replicas=2 \
  --set prometheus.enabled=true \
  --wait

cilium status --wait
echo "=== Cilium Installation Complete ==="
```

## Verify Installation

```bash
# Check status
cilium status

# Run connectivity test
cilium connectivity test

# Check Hubble
hubble status
hubble observe --last 100
```