# VPN User Management

## Create Users

```bash
#!/bin/bash
# scripts/create-vpn-user.sh

USERNAME="$1"
GROUP="${2:-dev}"  # admin or dev

HEADSCALE_POD=$(kubectl get pods -n headscale -l app=headscale -o jsonpath='{.items[0].metadata.name}')

# Create user
kubectl exec -n headscale ${HEADSCALE_POD} -- headscale users create ${USERNAME}

# Generate auth key
KEY=$(kubectl exec -n headscale ${HEADSCALE_POD} -- headscale preauthkeys create --user ${USERNAME} --reusable --expiration 7d 2>/dev/null | tail -1)

echo "User: ${USERNAME}"
echo "Group: ${GROUP}"
echo "Auth Key: ${KEY}"
echo ""
echo "Connect: tailscale up --login-server https://vpn.example.com --authkey ${KEY}"
```

## List Users

```bash
#!/bin/bash
# scripts/list-vpn-users.sh

HEADSCALE_POD=$(kubectl get pods -n headscale -l app=headscale -o jsonpath='{.items[0].metadata.name}')

echo "=== Users ==="
kubectl exec -n headscale ${HEADSCALE_POD} -- headscale users list

echo ""
echo "=== Nodes ==="
kubectl exec -n headscale ${HEADSCALE_POD} -- headscale nodes list
```

## Revoke Access

```bash
#!/bin/bash
# scripts/revoke-vpn-user.sh

USERNAME="$1"

if [[ -z "$USERNAME" ]]; then
  echo "Usage: $0 <username>"
  exit 1
fi

HEADSCALE_POD=$(kubectl get pods -n headscale -l app=headscale -o jsonpath='{.items[0].metadata.name}')

# Expire all auth keys
kubectl exec -n headscale ${HEADSCALE_POD} -- headscale preauthkeys list --user ${USERNAME}

# Delete all nodes for user
NODES=$(kubectl exec -n headscale ${HEADSCALE_POD} -- headscale nodes list --user ${USERNAME} -o json | jq -r '.[].id')

for node in $NODES; do
  kubectl exec -n headscale ${HEADSCALE_POD} -- headscale nodes delete --identifier ${node} --force
done

# Delete user
kubectl exec -n headscale ${HEADSCALE_POD} -- headscale users delete ${USERNAME} --force

echo "User ${USERNAME} revoked!"
```

## ACL Groups

Edit ACL to control access:

```json
{
  "groups": {
    "group:admin": ["user1", "user2"],
    "group:dev": ["dev1", "dev2", "dev3"],
    "group:readonly": ["viewer1"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    {
      "action": "accept",
      "src": ["group:dev"],
      "dst": [
        "tag:k8s:443",
        "tag:k8s:80",
        "tag:k8s:6443",
        "tag:gitlab:*"
      ]
    },
    {
      "action": "accept",
      "src": ["group:readonly"],
      "dst": [
        "tag:grafana:443",
        "tag:argocd:443"
      ]
    }
  ]
}
```

```bash
# Update ACL
kubectl edit configmap headscale-acl -n headscale

# Reload Headscale
kubectl rollout restart deployment headscale -n headscale
```