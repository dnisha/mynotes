---
longform:
  format: single
  title: VPC Endpoints
title: VPC Endpoints
---
**VPC Endpoint Features & AWS Solution Architect Associate Exam Revision Doc**

## What are VPC Endpoints?

**VPC Endpoints** provide private, direct connectivity between your Amazon VPC and supported AWS services, keeping all traffic inside the AWS network—without traversing the public internet[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).

![[VPC Endpoints Arch.png]]

## Types of VPC Endpoints

|Type|Description|Common Use-cases|Supported Services|
|---|---|---|---|
|**Gateway Endpoint**|Targets a specific AWS service. Route table is updated to direct traffic to the endpoint.|S3, DynamoDB access from private subnets|Amazon S3, Amazon DynamoDB[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)[9](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)|
|**Interface Endpoint (AWS PrivateLink)**|Elastic Network Interface (ENI) in your subnet, assigned private IPs.|Most AWS services (SNS, SQS, EC2 APIs, etc.), third-party SaaS|Many AWS and partner services[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)|
|**Gateway Load Balancer Endpoint**|Connects traffic to Gateway Load Balancers for security/inspection; used for advanced networking scenarios.|Network appliance integration, firewalls|GWLB (not usually in SAA exam)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)|

## Key Features and Benefits

- **Enhanced Security:** Traffic never leaves AWS, reducing exposure and attack surface. Minimizes risks of DDoS, data interception, or unauthorized access[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
- **Cost Optimization:** Avoids NAT Gateway and data transfer costs for S3/DynamoDB by staying within AWS; saves on internet egress fees[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)[8](https://www.clouddefense.ai/glossary/aws/vpc-endpoint).
    
- **Performance:** Lower latency and higher bandwidth than traversing internet; suitable for real-time and high-throughput workloads[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/).
    
- **Compliance:** Meets regulatory or compliance needs by avoiding public internet routes, keeping data within the AWS backbone[1](https://awsfundamentals.com/blog/vpc-endpoints)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/).
    
- **Simplified Network Management:** No complex firewall or NAT configuration, easier IAM integration and access control policies[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
- **High Availability:** VPC endpoints are horizontally scaled and redundant within your VPC[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
- **Fine-grained Access Control:** You can use VPC endpoint policies, IAM, and security groups for granular permissions[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/).

## Typical Exam Use-Cases

1. **Private Access to AWS Services:** S3 buckets need to be accessed from EC2 in private subnets—**gateway VPC endpoint** required[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[9](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html).
    
2. **Avoiding NAT Gateway Fees:** Applications reading/writing from S3 without public subnet or NAT[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
3. **Restricting Internet Exposure:** Service inside a VPC must not use public IPs to call SQS/SNS—**interface VPC endpoint** needed[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
4. **Compliance Requirement:** Customer data must not traverse the internet when accessing AWS services—use VPC endpoints[1](https://awsfundamentals.com/blog/vpc-endpoints)[3](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/).

## How VPC Endpoints Work

- **Gateway endpoints:** Update the VPC route table to add a route for the AWS service (S3/DynamoDB), directing it to the endpoint target[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[9](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html).
    
- **Interface endpoints:** Create an ENI in your subnet, with a private IP and security group; access AWS services privately using this ENI[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).
    
- **Gateway Load Balancer endpoints:** Specific for forwarding/inspecting traffic, not commonly used on the SAA exam[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/).

## Configuration Steps High-Level

1. **Create VPC Endpoint** (Gateway or Interface) in AWS Console or CLI.
    
2. **Configure Route Tables (Gateway)** or **Security Groups (Interface)** to control access.
    
3. **Optional:** Attach IAM policies for fine-grained service actions.

## Exam Revision Table

| Topic                      | Key Points                                                                       |
| -------------------------- | -------------------------------------------------------------------------------- |
| _Supported services_       | S3/DynamoDB: Gateway; Others: Interface                                          |
| _Traffic path_             | Remains within AWS global network                                                |
| _Security and compliance_  | No public internet; reduced attack surface; meet compliance requirements         |
| _Cost savings_             | Avoids NAT/internet gateway charges (especially for S3/DynamoDB); no egress fees |
| _Configuration_            | Gateway: Route tables; Interface: Security Groups                                |
| _Scalability/Availability_ | Horizontally scaled, redundant (managed by AWS)                                  |

## Notes & Limitations

- Not all AWS services support gateway endpoints—check exam questions for service compatibility[1](https://awsfundamentals.com/blog/vpc-endpoints)[2](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)[4](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)[9](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html).
    
- VPC endpoints are **region-specific**.
    
- Some limitations and design impacts exist (e.g., endpoint quota per VPC)[1](https://awsfundamentals.com/blog/vpc-endpoints).

For the AWS Solutions Architect Associate exam, know when and how to use **VPC Endpoints**—especially for **private S3/DynamoDB access**, **security**, and **cost optimization** scenarios.

1. [https://awsfundamentals.com/blog/vpc-endpoints](https://awsfundamentals.com/blog/vpc-endpoints)
2. [https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/](https://www.geeksforgeeks.org/devops/aws-vpc-endpoint/)
3. [https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/](https://www.learnaws.org/2023/09/05/aws-vpc-endpoints/)
4. [https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)
5. [https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)
6. [https://repost.aws/questions/QUnjR_kyowQPuPJQVxADEjYw/is-there-any-advantage-of-using-an-interface-vpc-endpoint-in-this-scenario](https://repost.aws/questions/QUnjR_kyowQPuPJQVxADEjYw/is-there-any-advantage-of-using-an-interface-vpc-endpoint-in-this-scenario)
7. [https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-access-to-vpc-private-endpoints.html](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-access-to-vpc-private-endpoints.html)
8. [https://www.clouddefense.ai/glossary/aws/vpc-endpoint](https://www.clouddefense.ai/glossary/aws/vpc-endpoint)
9. [https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)