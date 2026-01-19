---
longform:
  format: scenes
  title: Create Cluster Using EKSCTL
  sceneFolder: /
  scenes: []
  ignoredFiles: []
title: Create Cluster Using EKSCTL
---
### Step 0: Create the EC2 Keypair

Before running the Kubernetes setup, you must create the SSH key used in your instructions.

```bash

aws ec2 create-key-pair --key-name kube-demo --query 'KeyMaterial' --output text > kube-demo.pem

chmod 400 kube-demo.pem

```

### Step 1: Create the Cluster Configuration

Create a file named eks-cluster.yaml. This file incorporates **Step-01, Step-02, and Step-04** from your instructions.

```yaml

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksdemo1
  region: us-east-1
  version: "1.31"

# Step-02: Automatically associate IAM OIDC Provider
iam:
  withOIDC: true

managedNodeGroups:
  - name: eksdemo1-ng-public1
    instanceType: t3.medium
    minSize: 2
    maxSize: 4
    desiredCapacity: 2
    volumeSize: 20
    # Step-04: SSH Access
    ssh:
      allow: true
      publicKeyName: kube-demo
    # Step-04: IAM Add-on Policies
    iam:
      withAddonPolicies:
        autoScaler: true
        externalDNS: true
        imageBuilder: true # Full ECR access
        appMesh: true
        albIngress: true
        cloudWatch: true
        
```


### Step 2: Deploy the Cluster

Run the following command to create the VPC, Control Plane, OIDC Provider, and Node Groups all at once. This takes about 15–20 minutes.

```bash

eksctl create cluster -f eks-cluster.yaml

```

### Step 3: Verify the Setup (Step-05)

Once the deployment finishes, run these commands to verify:

**Check Nodes:**

```bash
kubectl get nodes -o wide
```

**Check NodeGroup details:**

```bash
eksctl get nodegroup --cluster eksdemo1 --region us-east-1
```


### Step 4: Update Security Group (Step-06)

To allow "All Traffic" on the worker nodes (Note: This is usually for testing; not recommended for production):

**Find the Node Security Group ID:**

```bash
NODE_SG=$(aws ec2 describe-security-groups --filters "Name=tag:Name,Values=eksctl-eksdemo1-nodegroup-eksdemo1-ng-public1*" --query "SecurityGroups[0].GroupId" --output text)
```

**Authorize All Inbound Traffic:**

```bash
aws ec2 authorize-security-group-ingress --group-id $NODE_SG --protocol all --port all --cidr 0.0.0.0/0
```

### Step 5: Log in to Worker Node

Find the public IP of one of your nodes and log in:

```bash
# Get Public IPs
kubectl get nodes -o wide

# SSH into a node
ssh -i kube-demo.pem ec2-user@<PUBLIC_IP_FROM_ABOVE>
```

### Summary of what was automated:

- **VPC Creation:** eksctl automatically created a VPC with Public and Private subnets.

- **Control Plane:** Initialized the Kubernetes Master nodes.

- **IAM OIDC:** Enabled so that Pods can assume IAM roles (using IRSA).

- **IAM Policies:** The withAddonPolicies section automatically attached the necessary AWS Managed Policies to your Node Instance Role (ALB Ingress, ECR, etc.).

- **Managed Node Group:** Created an ASG (Auto Scaling Group) that AWS manages for patching and updates.

### How to delete everything to avoid costs:

```bash
eksctl delete cluster --name eksdemo1 --region us-east-1
```