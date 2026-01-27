---
longform:
  format: single
  title: RBAC in AWS EKS
title: RBAC in AWS EKS
---

This document outlines Role-Based Access Control (RBAC) implementation in Amazon Elastic Kubernetes Service (EKS), focusing on the configuration example provided and its practical use cases.

## Prerequisites
- AWS CLI installed and configured
- kubectl installed
- EKS cluster created and kubeconfig configured
- IAM user/role with appropriate EKS permissions

## Configuration Example Analysis

### 1. **Namespace-scoped Role (`pod-reader`)**
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # Core API group (Pods, Services, etc.)
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

**Key Points:**
- Namespace: `rbac-test`
- Grants read-only access to pods and deployments
- Supports both legacy (`extensions`) and current (`apps`) API groups

### 2. **RoleBinding (`read-pods`)**
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Key Points:**
- Binds the `pod-reader` role to user `dev-user`
- Effective only in `rbac-test` namespace

### 3. **AWS Auth ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::158311564815:user/dev-user
      username: dev-user
```

**Key Points:**
- Maps AWS IAM user to Kubernetes user
- Stored in `kube-system` namespace
- Required for EKS IAM integration

## Implementation Steps

### Step 1: Create Namespace
```bash
kubectl create namespace rbac-test
```

### Step 2: Apply RBAC Configuration
1. **Create Role and RoleBinding:**
```bash
kubectl apply -f rbac-config.yaml -n rbac-test
```

2. **Verify Role creation:**
```bash
kubectl get role -n rbac-test
kubectl describe role pod-reader -n rbac-test
```

3. **Verify RoleBinding:**
```bash
kubectl get rolebinding -n rbac-test
kubectl describe rolebinding read-pods -n rbac-test
```

### Step 3: Configure AWS Authentication
1. **Update aws-auth ConfigMap:**
```bash
kubectl apply -f aws-auth-configmap.yaml -n kube-system
```

2. **Alternatively, edit existing ConfigMap:**
```bash
kubectl edit configmap aws-auth -n kube-system
```

### Step 4: Test Access
1. **As dev-user, test pod access:**
```bash
# This assumes you've configured AWS credentials for dev-user
kubectl get pods -n rbac-test
```

2. **Test deployment access:**
```bash
kubectl get deployments -n rbac-test
```

3. **Verify restricted access (should fail):**
```bash
kubectl get pods -n default  # Should fail - no access to default namespace
kubectl delete pod <pod-name> -n rbac-test  # Should fail - no delete permission
```

## Common Use Cases

### Use Case 1: **Development Team Access**
**Scenario:** Development team needs read-only access to their namespace for debugging and monitoring.

**Configuration:**
```yaml
# dev-team-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: dev-team-namespace
  name: dev-team-viewer
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "jobs", "configmaps"]
  verbs: ["get", "list", "watch"]
```

### Use Case 2: **CI/CD Pipeline Service Account**
**Scenario:** Jenkins/GitLab Runner needs deployment permissions in specific namespaces.

**Configuration:**
```yaml
# cicd-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: staging
  name: cicd-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### Use Case 3: **Cross-Namespace Admin**
**Scenario:** Team lead needs admin access across multiple project namespaces.

**Configuration:**
```yaml
# team-lead-clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: project-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# ClusterRoleBinding for specific namespaces
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: team-lead-admin
  namespace: project-a
subjects:
- kind: User
  name: team-lead-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: project-admin
  apiGroup: rbac.authorization.k8s.io
```

### Use Case 4: **Read-Only Auditor Role**
**Scenario:** Security team needs read access across all namespaces for compliance auditing.

**Configuration:**
```yaml
# auditor-clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-auditor
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["*"]
  verbs: ["get", "list"]

# Cluster-wide binding
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: security-auditors
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-auditor
  apiGroup: rbac.authorization.k8s.io
```

## Best Practices

### 1. **Principle of Least Privilege**
- Grant only necessary permissions
- Use specific verbs instead of `["*"]`
- Scope to specific namespaces when possible

### 2. **Use ServiceAccounts for Applications**
```yaml
# Instead of IAM user for applications
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: app-namespace
```

### 3. **Regular Auditing**
```bash
# Check what a user can do
kubectl auth can-i --list --as=dev-user -n rbac-test

# Check specific permissions
kubectl auth can-i create deployments --as=dev-user -n rbac-test
```

### 4. **Organize with Groups**
```yaml
# AWS ConfigMap with groups
mapUsers: |
  - userarn: arn:aws:iam::123456789012:user/dev1
    username: dev1
    groups:
      - developers
  - userarn: arn:aws:iam::123456789012:user/dev2
    username: dev2
    groups:
      - developers

# Then bind to groups
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
```

### 5. **Separate IAM Roles for Different Environments**
```yaml
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/EKS-DevAdmin
    username: dev-admin
    groups:
      - system:masters
  - rolearn: arn:aws:iam::123456789012:role/EKS-DevView
    username: dev-view
    groups:
      - developers
```

## Troubleshooting

### Common Issues and Solutions:

1. **"Error from server (Forbidden)"**
   ```bash
   # Check permissions
   kubectl auth can-i <verb> <resource> --as=<user> -n <namespace>
   
   # Check role bindings
   kubectl get rolebinding,clusterrolebinding -A | grep <user>
   ```

2. **AWS IAM User Not Recognized**
   ```bash
   # Verify aws-auth ConfigMap
   kubectl get configmap aws-auth -n kube-system -o yaml
   
   # Check IAM user ARN format
   # Correct: arn:aws:iam::<account-id>:user/<username>
   ```

3. **Permissions Not Working Across Namespaces**
   - Use ClusterRole and ClusterRoleBinding for cross-namespace access
   - Verify namespace specification in Role/RoleBinding