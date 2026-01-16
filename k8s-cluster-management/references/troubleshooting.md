# Troubleshooting

Common issues and solutions. Run all commands from bastion server.

## Kubespray Installation Issues

### SSH Connection Failed

```bash
# Check SSH access from bastion to nodes
ssh ${PROJECT}-master-1 hostname
ssh ${PROJECT}-worker-1 hostname

# Verify SSH key
ls -la ~/.ssh/id_rsa

# Check bastion can reach private network
ping 10.0.1.1
```

### Ansible Timeout

```bash
# Increase timeout in ansible.cfg
[defaults]
timeout = 60

# Or pass as environment variable
export ANSIBLE_TIMEOUT=60
```

### Python Dependencies

```bash
# Recreate venv
cd kubespray
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -U pip
pip install -r requirements.txt
```

## Cluster Issues

### Nodes Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check Cilium
cilium status
kubectl get pods -n kube-system | grep cilium

# Restart Cilium if needed
kubectl rollout restart daemonset/cilium -n kube-system
```

### Pod Stuck Pending

```bash
# Check events
kubectl describe pod <pod-name> -n <namespace>

# Check resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check PVC if using storage
kubectl get pvc -A
```

### DNS Not Working

```bash
# Test DNS
kubectl run -it --rm debug --image=busybox -- nslookup kubernetes

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## Cilium Issues

### Connectivity Test Fails

```bash
# Run connectivity test
cilium connectivity test

# Check Cilium status
cilium status --verbose

# Check Hubble
hubble status
hubble observe --last 100
```

### Gateway Not Working

```bash
# Check GatewayClass
kubectl get gatewayclass

# Check Gateway
kubectl get gateway -A
kubectl describe gateway <name> -n <namespace>

# Check HTTPRoute
kubectl get httproute -A
```

## cert-manager Issues

### Certificate Not Issued

```bash
# Check certificate status
kubectl get certificates -A
kubectl describe certificate <name> -n <namespace>

# Check certificate request
kubectl get certificaterequest -A

# Check challenges (for ACME)
kubectl get challenges -A
kubectl describe challenge <name> -n <namespace>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

### DNS01 Challenge Failing

```bash
# Check ClusterIssuer
kubectl describe clusterissuer letsencrypt-prod

# Check DNS propagation
dig _acme-challenge.example.com TXT

# Check Hetzner DNS token
kubectl get secret hetzner-dns-token -n cert-manager
```

## MetalLB Issues

### No External IP

```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check IP pool
kubectl get ipaddresspool -n metallb-system

# Check L2 advertisement
kubectl get l2advertisement -n metallb-system

# Check service
kubectl describe svc <service-name> -n <namespace>
```

## Logs

```bash
# System pods
kubectl logs -n kube-system <pod-name>

# Follow logs
kubectl logs -f -n <namespace> <pod-name>

# Previous container logs
kubectl logs -n <namespace> <pod-name> --previous

# All containers in pod
kubectl logs -n <namespace> <pod-name> --all-containers
```