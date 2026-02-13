```markdown
The document is hosted at - [https://dnisha.github.io/mynotes/](https://dnisha.github.io/mynotes/)


```

**Create Cluster (Step 1.1)**
- **Files:** [01-infrastructure/kafka-cluster.yaml](01-infrastructure/kafka-cluster.yaml)
- **Purpose:** Provision an EKS cluster for Kafka with nodes in three private subnets and required addons (including EBS CSI driver).
- **Prerequisites:** AWS CLI configured with the correct profile/region, `eksctl` installed, and `jq` (optional) for JSON parsing.
- **1 — Create VPC and subnets (AWS CLI):** Run these commands and save the resulting IDs. Replace CIDRs and names as needed.

```bash
# create VPC and capture the VPC id
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kafka-vpc}]' --query 'Vpc.VpcId' --output text)
echo "VPC_ID=$VPC_ID"

# create private subnets in three AZs (example CIDRs)
SUBNET_PRIVATE_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-west-2a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-a}]' --query 'Subnet.SubnetId' --output text)
SUBNET_PRIVATE_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-west-2b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-b}]' --query 'Subnet.SubnetId' --output text)
SUBNET_PRIVATE_C=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-west-2c --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-c}]' --query 'Subnet.SubnetId' --output text)
echo "$SUBNET_PRIVATE_A $SUBNET_PRIVATE_B $SUBNET_PRIVATE_C"

# create public subnets (for load balancers) and associate route table/IGW as required
SUBNET_PUBLIC_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.101.0/24 --availability-zone us-west-2a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-a}]' --query 'Subnet.SubnetId' --output text)
SUBNET_PUBLIC_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.102.0/24 --availability-zone us-west-2b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-b}]' --query 'Subnet.SubnetId' --output text)
SUBNET_PUBLIC_C=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.103.0/24 --availability-zone us-west-2c --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-c}]' --query 'Subnet.SubnetId' --output text)
echo "$SUBNET_PUBLIC_A $SUBNET_PUBLIC_B $SUBNET_PUBLIC_C"

# (Optional) Create and attach an Internet Gateway, route tables, and enable public IP assignment for public subnets.
```

- **2 — Update the cluster config:** Edit the file above to insert the VPC ID and the three private subnet IDs into the `vpc.id` and `vpc.subnets.private` fields, and (optionally) update public subnet IDs.
- **3 — Create the EKS cluster with eksctl:**

```bash
# from the repository root
eksctl create cluster -f 01-infrastructure/kafka-cluster.yaml
```

- **Notes:**
  - The `managedNodeGroups` defined in the YAML put nodes in the private subnets (`privateNetworking: true`) and attach EBS permissions for the CSI driver.
  - Ensure `iam.withOIDC: true` is acceptable for your account; this enables IRSA for addons.
  - After cluster creation, verify the `aws-ebs-csi-driver` addon is present and healthy (`kubectl get pods -n kube-system`).

Replace the placeholder IDs in the YAML with the IDs produced by the AWS CLI steps above if you are creating new VPC/subnets.
