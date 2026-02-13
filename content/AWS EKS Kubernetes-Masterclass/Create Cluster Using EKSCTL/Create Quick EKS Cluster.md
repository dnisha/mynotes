---
longform:
  format: single
  title: Create Production Grade Cluster
title: Create Production Grade Cluster
---
# **EKS Cluster Setup**

## **Prerequisites**
- AWS CLI configured
- `eksctl` and `kubectl` installed

## **Quick Deployment (10 min)**

### **Step 1: Create Cluster Configuration**
Create `eks-prod.yaml`:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cluster
  region: ap-south-1
  version: "1.33"

iam:
  withOIDC: true

managedNodeGroups:
  - name: standard-workers
    instanceType: c7i-flex.large
    # Note : change th instance type that latest aws eks supports , else node will not get provisioned
    desiredCapacity: 2
    minSize: 2
    maxSize: 3
    volumeSize: 100
    volumeType: gp3
    privateNetworking: true
    ssh:
      allow: false  # Disable SSH for production
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator"]

```

### **Step 2: Deploy Cluster**
```bash
# Single command creates everything
eksctl create cluster -f eks-prod.yaml

# Configure kubectl
aws eks update-kubeconfig --region ap-south-1 --name prod-cluster
```

### **Step 3: Verify**
```bash
# Check cluster status
kubectl get nodes
kubectl get pods -A
eksctl get nodegroup --cluster prod-cluster
```

## **Production Features**
- **Private workers** - Enhanced security
- **Multi-AZ** - High availability
- **Auto-scaling** - 3-6 nodes
- **OIDC** - IAM roles for pods
- **Audit logging** - CloudWatch enabled
- **gp3 volumes** - Better performance

## **Cleanup**
```bash
eksctl delete cluster --name prod-cluster --region ap-south-1
```