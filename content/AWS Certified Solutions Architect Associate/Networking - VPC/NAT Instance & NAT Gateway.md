---
longform:
  format: single
  title: NAT Instance & NAT Gateway
title: NAT Instance & NAT Gateway
---
The main differences between **NAT instances** and **NAT gateways** in AWS relate to **availability, scalability, management, and cost**:

|Attribute|NAT Gateway|NAT Instance|
|---|---|---|
|**Availability**|Highly available, redundant within each AZ. AWS manages HA.|No built-in HA; you must manage failover via scripts or manually.|
|**Bandwidth**|Scales automatically up to 100 Gbps.|Limited by EC2 instance type and size; requires manual scaling.|
|**Management**|Fully managed by AWS; no maintenance required.|User-managed: You must maintain OS, patches, software updates.|
|**Performance**|Optimized software specifically for NAT traffic.|Generic EC2 AMI configured as NAT; less optimized.|
|**Cost**|Charged based on hours of usage and data processed; generally higher.|Cheaper instance costs but increased operational overhead.|
|**Type/Size**|Fixed offering; no need to choose instance size.|Must pick instance type and size according to traffic needs.|
|**Public IP**|Elastic IP assigned at creation; cannot be changed later.|Elastic IP or public IP can be associated or reassigned.|
|**Security Groups**|Cannot associate security groups directly. Use subnet-level ACLs and security groups on backend resources.|Can assign security groups to control inbound/outbound traffic on the instance.|
|**Port Forwarding**|Not supported.|Supported via manual configuration.|
|**Use as Bastion**|Not supported.|Can be used as a bastion host.|
|**Traffic Metrics**|Supports CloudWatch metrics.|CloudWatch metrics available for the EC2 instance.|
|**Timeout Behavior**|Returns TCP RST on connection timeout.|Sends TCP FIN to gracefully close connections.|
|**IP Fragmentation**|Supports UDP fragmentation; TCP and ICMP fragments are dropped.|Supports reassembly for UDP, TCP, and ICMP fragments.|

**Key recommendations:**

- AWS _recommends NAT gateways_ for most workloads due to their better availability, automatic scaling, ease of maintenance, and optimized performance[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)5[6](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html).
    
- NAT instances provide more control and flexibility (e.g., port forwarding, bastion hosts) but require ongoing manual management and may be harder to scale or maintain[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)[3](https://www.linkedin.com/pulse/aws-nat-gateway-instance-simple-guide-enthusiasts-pi%C3%B1ero-estrada-1zmce).
    
- NAT gateways automatically handle failover and redundancy within an Availability Zone, but best practice is to deploy one in each AZ for zone-independent architecture[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html).
    
- NAT instances require you to manage failover, often through scripting or custom setups, and handle all maintenance tasks like patching and monitoring[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)[2](https://www.reddit.com/r/aws/comments/yb7nca/whats_the_difference_between_a_nat_instance_and_a/).
    
- NAT gateways cost more per data processed but reduce admin overhead and support up to 100 Gbps; NAT instances’ bandwidth depends on the EC2 instance and can bottleneck under heavy loads[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)[4](https://jayendrapatil.com/tag/nat-gateway-vs-nat-instance/).
    

In general, NAT gateways suit **production, high-availability, and scalable environments**, while NAT instances might be chosen for **cost-sensitive, legacy, or highly customized scenarios**.

If migrating from a NAT instance to a NAT gateway, AWS provides a simple process to replace the NAT route and reassign Elastic IPs, with caution about connection drops during the switch[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html).

Thus, the **NAT gateway is the superior, fully managed, scalable solution whereas the NAT instance is a manual, customizable but less resilient option**. The choice depends on your needs for control versus ease of use.