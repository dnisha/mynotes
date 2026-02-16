---
longform:
  format: single
  title: AWS Load Balancer Controller Installation
title: AWS Load Balancer Controller Installation
---

## Architecture Overview

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for Kubernetes clusters. Below is the architecture diagram showing how the controller integrates with EKS and AWS services:


## Prerequisites

- AWS CLI configured with appropriate permissions
- `kubectl` configured to access your EKS cluster
- `helm` v3 installed
- EKS cluster with OIDC provider enabled

## Step-by-Step Installation Guide

### Step 1: Download IAM Policy

Download the IAM policy file required for the AWS Load Balancer Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.0/docs/install/iam_policy.json
```

### Step 2: Create IAM Policy

Create an IAM policy using the downloaded file:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

**Note the Policy ARN** from the output - you'll need it later.

### Step 3: Get OIDC Provider ARN

Get your EKS cluster's OIDC provider ARN:

```bash
# Replace with your cluster name and region
CLUSTER_NAME=your-cluster-name
REGION=ap-south-1

aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --region $REGION \
    --query "cluster.identity.oidc.issuer" \
    --output text
```

The output will be something like: `https://oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXXXX`

Extract the OIDC ID and construct the ARN:
```
OIDC_ARN=arn:aws:iam::<account-id>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXXXX
```

### Step 4: Create Trust Policy File

Create a `trust-policy.json` file:

```bash
cat > trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "${OIDC_ARN}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "$(echo ${OIDC_ARN#*:oidc-provider/}):sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
                }
            }
        }
    ]
}
EOF
```

**Note:** Replace `${OIDC_ARN}` with your actual OIDC ARN.

### Step 5: Create IAM Role

Create the IAM role with the trust policy:

```bash
aws iam create-role \
    --role-name AWSLoadBalancerControllerRole \
    --assume-role-policy-document file://trust-policy.json \
    --region ap-south-1
```

### Step 6: Attach Policy to Role

Attach the IAM policy to the role:

```bash
# Replace with your actual policy ARN
POLICY_ARN=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy

aws iam attach-role-policy \
    --role-name AWSLoadBalancerControllerRole \
    --policy-arn $POLICY_ARN \
    --region ap-south-1
```

### Step 7: Create Kubernetes Service Account

Create a ServiceAccount with the IAM role annotation:

```bash
cat > service-account.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/AWSLoadBalancerControllerRole
EOF

kubectl apply -f service-account.yaml
```

### Step 8: Install Controller using Helm

Add and update the EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the AWS Load Balancer Controller:

```bash
# Replace with your cluster name and VPC ID
CLUSTER_NAME=your-cluster-name
VPC_ID=vpc-xxxxxxxx

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$VPC_ID \
  --version 1.13.0
```

### Step 9: Verify Installation

Check if the controller pods are running:

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/instance=aws-load-balancer-controller
```

Check the controller logs:

```bash
kubectl -n kube-system logs -f --selector app.kubernetes.io/instance=aws-load-balancer-controller
```

## Verification and Testing

### Test with a Sample Application

Create a test deployment and service:

```bash
cat > test-app.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-app
  namespace: default
spec:
  selector:
    app: test-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-app
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-app
            port:
              number: 80
EOF

kubectl apply -f test-app.yaml
```

Check if the ALB is provisioned:

```bash
kubectl get ingress -n default -w
```

## Troubleshooting

### Common Issues and Solutions

1. **Controller pods not starting:**
   ```bash
   # Check pod events
   kubectl -n kube-system describe pod -l app.kubernetes.io/instance=aws-load-balancer-controller
   
   # Check IAM role permissions
   aws iam simulate-principal-policy \
       --policy-source-arn arn:aws:iam::<account-id>:role/AWSLoadBalancerControllerRole \
       --action-names elasticloadbalancing:CreateLoadBalancer
   ```

2. **Ingress not provisioning ALB:**
   ```bash
   # Check controller logs
   kubectl -n kube-system logs -f --selector app.kubernetes.io/instance=aws-load-balancer-controller
   
   # Check ingress events
   kubectl describe ingress <ingress-name>
   ```

3. **Subnet tags missing:**
   Ensure your subnets have the correct tags:
   ```bash
   # For public subnets
   aws ec2 create-tags --resources subnet-xxx --tags Key=kubernetes.io/role/elb,Value=1
   
   # For private subnets
   aws ec2 create-tags --resources subnet-yyy --tags Key=kubernetes.io/role/internal-elb,Value=1
   ```

## Cleanup

To remove the AWS Load Balancer Controller:

```bash
# Uninstall Helm chart
helm uninstall aws-load-balancer-controller -n kube-system

# Delete ServiceAccount
kubectl delete -f service-account.yaml

# Detach and delete IAM role
aws iam detach-role-policy \
    --role-name AWSLoadBalancerControllerRole \
    --policy-arn $POLICY_ARN

aws iam delete-role --role-name AWSLoadBalancerControllerRole

# Delete IAM policy
aws iam delete-policy --policy-arn $POLICY_ARN

# Delete downloaded files
rm -f iam_policy.json trust-policy.json service-account.yaml test-app.yaml
```

## References

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)