# Essential Components

Core components installed with every Kubernetes cluster via Kubespray.

## Installed by Kubespray

| Component | Purpose |
|-----------|--------|
| containerd | Container runtime |
| Cilium | CNI networking |
| CoreDNS | Cluster DNS |
| metrics-server | Resource metrics |
| NodeLocal DNS | DNS caching |

## Post-Installation

| Component | Purpose | Script |
|-----------|---------|--------|
| Gateway API | Ingress routing | `install-gateway-api.sh` |
| MetalLB | LoadBalancer IPs | `install-metallb.sh` |
| cert-manager | TLS automation | `install-cert-manager.sh` |

## Version Matrix

| Component | Version |
|-----------|--------|
| Kubernetes | v1.34.3 |
| Kubespray | v2.26.0 |
| Cilium | v1.18.5 |
| Gateway API | v1.4.0 |
| cert-manager | v1.19.2 |
| MetalLB | v0.14.8 |

## Verification

```bash
# Run from bastion
kubectl get nodes -o wide
kubectl get pods -A
cilium status
kubectl get gatewayclass
```