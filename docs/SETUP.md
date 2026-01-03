# Kubernetes RBAC Implementation - Setup Guide

Complete setup guide for implementing enterprise-grade RBAC in Kubernetes clusters.

## Prerequisites

- **Kubernetes cluster** (v1.28+)
- **kubectl** configured with cluster-admin access
- **Network Policy support** (CNI plugin: Calico, Cilium, or Weave)
- **Basic Kubernetes knowledge** (pods, deployments, services)

## Quick Start (10 minutes)

```bash
# 1. Clone repository
git clone https://github.com/vanHeemstraSystems/kubernetes-rbac-implementation.git
cd kubernetes-rbac-implementation

# 2. Run setup script
./scripts/setup-namespaces.sh

# 3. Verify RBAC configuration
./scripts/verify-rbac.sh

# 4. Generate test kubeconfig
./scripts/generate-kubeconfig.sh team-a-developer team-a-dev

# 5. Test with limited permissions
kubectl --kubeconfig=team-a-developer-kubeconfig.yaml get pods
```

## Detailed Setup

### Step 1: Create Namespaces

```bash
kubectl apply -f manifests/namespaces/
```

This creates:

- `team-a-dev` - Team A development environment
- `team-a-prod` - Team A production environment
- `team-b-dev` - Team B development environment
- `shared-services` - Shared infrastructure

Verify:

```bash
kubectl get namespaces -l managed-by=rbac-lab
```

### Step 2: Deploy Roles

**Developer Role** (development namespaces):

```bash
kubectl apply -f manifests/roles/developer-role.yaml
```

Permissions:

- ✅ Create/update/delete: Deployments, Pods, Services, ConfigMaps
- ✅ Read: Secrets (read-only)
- ✅ Exec into pods, view logs
- ❌ Modify: RBAC, ResourceQuotas, NetworkPolicies

**Deployer Role** (production namespaces):

```bash
kubectl apply -f manifests/roles/deployer-role.yaml
```

Permissions:

- ✅ Update: Deployments, ConfigMaps
- ✅ Read: All resources
- ❌ Create new resources
- ❌ Delete existing resources
- ❌ Exec into pods (security)

**Viewer Role** (all namespaces):

```bash
kubectl apply -f manifests/roles/viewer-role.yaml
```

Permissions:

- ✅ Read: Most resources
- ❌ View: Secrets
- ❌ Write: Any resources

### Step 3: Create Service Accounts

```bash
kubectl apply -f manifests/serviceaccounts/
```

Service accounts created:

- `team-a-developer` (team-a-dev namespace)
- `team-a-deployer` (team-a-prod namespace)
- `team-a-viewer` (team-a-prod namespace)
- `team-b-developer` (team-b-dev namespace)

Verify:

```bash
kubectl get sa -A -l role
```

### Step 4: Create RoleBindings

```bash
kubectl apply -f manifests/rolebindings/
```

This binds:

- ServiceAccounts → Roles (within namespace)

Verify:

```bash
kubectl get rolebindings -A
```

### Step 5: Apply Network Policies

```bash
# Apply to each namespace
for ns in team-a-dev team-a-prod team-b-dev; do
    kubectl apply -f manifests/networkpolicies/default-deny.yaml -n $ns
    kubectl apply -f manifests/networkpolicies/allow-same-namespace.yaml -n $ns
done
```

This implements:

- **Default deny**: All traffic blocked by default
- **Same namespace**: Pods can communicate within namespace
- **Cross-namespace isolation**: Teams cannot access each other’s pods

Verify:

```bash
kubectl get networkpolicies -A
```

## Verification

### Automated Verification

```bash
./scripts/verify-rbac.sh
```

Expected output:

```
✓ team-a-developer in team-a-dev CAN create deployments
✓ team-a-developer in team-a-dev CANNOT create secrets
✓ team-a-deployer in team-a-prod CAN update deployments
✓ team-a-deployer in team-a-prod CANNOT delete deployments
✓ team-a-viewer in team-a-prod CAN get pods
✓ team-a-viewer in team-a-prod CANNOT create deployments
```

### Manual Verification

Test developer permissions:

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:team-a-dev:team-a-developer \
  -n team-a-dev
# Output: yes

kubectl auth can-i delete resourcequotas \
  --as=system:serviceaccount:team-a-dev:team-a-developer \
  -n team-a-dev
# Output: no
```

Test cross-namespace isolation:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:team-a-dev:team-a-developer \
  -n team-b-dev
# Output: no
```

## Using Service Accounts

### Generate Kubeconfig for User

```bash
./scripts/generate-kubeconfig.sh team-a-developer team-a-dev
```

This creates: `team-a-developer-kubeconfig.yaml`

### Test with Limited Permissions

```bash
# Set kubeconfig
export KUBECONFIG=team-a-developer-kubeconfig.yaml

# Should succeed
kubectl get pods
kubectl create deployment nginx --image=nginx

# Should fail
kubectl get pods -n team-b-dev  # Cross-namespace
kubectl delete resourcequota -n team-a-dev  # Not permitted
```

## Production Deployment

### 1. Integrate with Identity Provider

For enterprise SSO (Azure AD, Okta, etc.):

```bash
# Example: Azure AD integration
kubectl apply -f - <<EOF
apiVersion: v1
kind: Config
users:
- name: alice@company.com
  user:
    auth-provider:
      name: oidc
      config:
        client-id: <client-id>
        idp-issuer-url: https://login.microsoftonline.com/<tenant-id>/v2.0
        id-token: <token>
EOF
```

### 2. Enable Audit Logging

Add to API server flags:

```yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

Audit policy:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
```

### 3. Set Resource Quotas

Per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-dev-quota
  namespace: team-a-dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
```

### 4. Monitoring and Alerts

Monitor RBAC changes:

```bash
kubectl get events --field-selector involvedObject.kind=RoleBinding -A -w
```

## Common Use Cases

### Use Case 1: Onboard New Developer

```bash
# 1. Create service account
kubectl create sa alice -n team-a-dev

# 2. Bind to developer role
kubectl create rolebinding alice-developer \
  --role=developer \
  --serviceaccount=team-a-dev:alice \
  -n team-a-dev

# 3. Generate kubeconfig
./scripts/generate-kubeconfig.sh alice team-a-dev

# 4. Send kubeconfig to developer
# Time: 5 minutes
```

### Use Case 2: Production Deploy Access

```bash
# 1. Create deployer service account
kubectl create sa bob -n team-a-prod

# 2. Bind to deployer role (limited permissions)
kubectl create rolebinding bob-deployer \
  --role=deployer \
  --serviceaccount=team-a-prod:bob \
  -n team-a-prod

# 3. Generate kubeconfig
./scripts/generate-kubeconfig.sh bob team-a-prod
```

### Use Case 3: Read-Only Access for Manager

```bash
# 1. Create viewer service account
kubectl create sa manager -n team-a-prod

# 2. Bind to viewer role
kubectl create rolebinding manager-viewer \
  --role=viewer \
  --serviceaccount=team-a-prod:manager \
  -n team-a-prod

# 3. Generate kubeconfig
./scripts/generate-kubeconfig.sh manager team-a-prod
```

## Troubleshooting

### Permission Denied

```bash
# Check what user can do
kubectl auth can-i --list \
  --as=system:serviceaccount:team-a-dev:team-a-developer \
  -n team-a-dev

# Check role binding
kubectl get rolebinding -n team-a-dev
kubectl describe rolebinding team-a-developer-binding -n team-a-dev
```

### Service Account Not Working

```bash
# Verify service account exists
kubectl get sa -n team-a-dev

# Verify token secret exists
kubectl get secrets -n team-a-dev | grep team-a-developer

# Check role binding
kubectl get rolebinding -n team-a-dev team-a-developer-binding
```

### Network Policy Issues

```bash
# Check if CNI supports network policies
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"

# Verify network policies applied
kubectl get networkpolicies -A

# Test connectivity
kubectl run test-pod --image=busybox -n team-a-dev -- sleep 3600
kubectl exec -n team-a-dev test-pod -- wget -O- http://service.team-b-dev
# Should fail due to network policy
```

## Security Best Practices

1. **Least Privilege**: Start with minimal permissions, add as needed
1. **Regular Audits**: Review RBAC monthly, remove unused access
1. **Separation of Duties**: Different roles for dev/prod
1. **Network Isolation**: Always use NetworkPolicies
1. **Audit Logging**: Enable for compliance and forensics
1. **Secret Management**: Use external secret stores (Vault, Azure KeyVault)
1. **Pod Security**: Enable Pod Security Standards
1. **Monitoring**: Alert on RBAC changes

## Next Steps

- **Advanced**: Implement custom admission webhooks
- **Automation**: Integrate with CI/CD for GitOps RBAC
- **Scale**: Add more teams and environments
- **Compliance**: Configure audit logging for SOC2/ISO27001

-----

**Version**: 1.0  
**Last Updated**: January 2026
