---
longform:
  format: single
  title: Transit Gateway
title: Transit Gateway
---
# AWS Transit Gateway: Overview, Architectures, Problems Solved, and Multi-Account Sharing

## What is AWS Transit Gateway?

**AWS Transit Gateway (TGW)** is a cloud-managed network hub designed to simplify network connectivity across **multiple VPCs, on-premises resources, and remote networks**. It acts as a scalable, centralized router at Layer 3, managing the flow of traffic between attached resources, such as VPCs, VPNs, AWS Direct Connect gateways, and other transit gateways[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/)[2](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html).

![[Transite Gateway Arch.png]]

## Problems AWS Transit Gateway Solves

- **Complex VPC Peering:** Eliminates the exponential complexity of mesh VPC peering by serving as a centralized hub, enabling a dramatic reduction in the number of required peering connections[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/).
    
- **Operational Overhead:** Centralizes routing, monitoring, and security, reducing manual configuration and errors in large environments.
    
- **Scalability Constraints:** Supports thousands of connections and route entries, far exceeding traditional limits imposed by direct peering.
    
- **Inefficient Routing:** Replaces inefficient mesh or chained routing with optimized hub-and-spoke traffic flow.
    
- **Fragmented Security and Monitoring:** Enables centralized enforcement and visibility for all east-west and hybrid traffic flows.

## Core Architecture Patterns with AWS Transit Gateway

## 1. Hub-and-Spoke

- **Central hub** (TGW) connects to multiple **spokes** (VPCs, VPNs, or Direct Connects).
- Spokes communicate via the hub, **not directly with each other**, greatly reducing connectivity complexity[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/)[2](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html).

**Use Cases:**

- Enterprises managing tens to hundreds of VPCs.
- Organizations requiring centralized policy control and auditing.

## 2. Multi-Region/Inter-Region Peering

- **Transit Gateways in different AWS regions** are peered to extend network connectivity globally.
- VPCs in different regions can communicate via their regional TGWs over AWS’s backbone[3](https://aws.amazon.com/blogs/networking-and-content-delivery/aws-cloud-wan-and-aws-transit-gateway-migration-and-interoperability-patterns/).

**Use Cases:**

- Global applications.
- Disaster recovery requiring cross-region VPC communication.

## 3. Hybrid Connectivity

- **On-premises environments** connect to AWS via VPN or Direct Connect attachments to the TGW.
- Offers seamless **hybrid cloud** integration and migration paths for legacy data centers[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/).

**Use Cases:**

- Business continuity.
- Gradual migration to cloud.

## 4. Multi-Account Topology

- **TGW is shared across multiple AWS accounts** (using AWS Resource Access Manager).
- Central TGW allows unified governance and segmentation while permitting account-level autonomy[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/).

**Use Cases:**

- Enterprise with micro-segmented accounts for business units or environments.
- SaaS providers or MSPs managing multi-tenant networking.

## 5. Segmented and Isolated Network

- Multiple **TGW route tables** isolate or segment network traffic (e.g. environment isolation between dev/prod/shared).
- Provides VRF-like network segmentation[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/)[2](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html).

**Use Cases:**

- Strict regulatory/compliance isolation needs.
- Egress controls (centralized or restricted internet access).

## 6. Egress-Only or Centralized Network Services

- Shared services like DNS, security inspection, or internet access are centralized in one or more VPCs, available across the environment via TGW[4](https://github.com/aws-samples/aws-transit-gateway-egress-vpc-pattern/blob/master/README.md).

**Use Cases:**

- Platform teams consolidating controls such as outbound internet or threat inspection.

## 7. Third-Party/Partner Connectivity

- TGW facilitates secure connection to external partner networks or managed service providers, supporting B2B or SaaS integrations[5](https://docs.aws.amazon.com/prescriptive-guidance/latest/integrate-third-party-services/architecture-3.html).

## AWS Transit Gateway Architecture Diagrams

Below are illustrative architecture diagrams based on typical AWS and reference documentation:

## Hub-and-Spoke Model

> VPC A ----  
> TGW ------- VPC C  
> VPC B ----/ |  
> |--- VPN/Direct Connect (On-premises)

---

## Multi-Region/Inter-Region Peering

> [Region 1] [Region 2]  
> TGW-1 ---- VPCs --- TGW Peering --- TGW-2 ---- VPCs

---

## Multi-Account Connectivity

> [Account 1: VPCs]---|  
> TGW (shared via RAM)---[Account 2: VPCs]  
> [Account 3: VPCs]---|

---

## Centralized Egress/Shared Services

> [Private VPCs]--- TGW --- [Egress/Inspection VPC] --- Internet

## How to Share AWS Transit Gateway Across Accounts

**AWS Resource Access Manager (RAM)** is used to share a Transit Gateway:

- The TGW owner shares the resource with other accounts or AWS Organization units.

- Recipient accounts can then **attach their VPCs** to the shared TGW, enabling cross-account network integration.

- The owner controls the Transit Gateway, **route tables, policies, and monitoring**, while attached accounts manage their own connections. Attachments can be individually removed by either party[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/).
## Summary Table: Key Features

|Feature|Description|
|---|---|
|Centralized Routing|All traffic managed via a central hub|
|High Scalability|Thousands of attachments/connections supported|
|Multi-Account Support|Share TGW using AWS RAM for unified enterprise access|
|Route Table Segmentation|Fine-grained policy, isolation and isolation|
|Hybrid/Global Reach|VPN, Direct Connect, Inter-region peering|
|Traffic Inspection|Centralized egress, logging, and service integration|

**References:**

- AWS Documentation and Architecture Guides[1](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/)[2](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)[3](https://aws.amazon.com/blogs/networking-and-content-delivery/aws-cloud-wan-and-aws-transit-gateway-migration-and-interoperability-patterns/)[4](https://github.com/aws-samples/aws-transit-gateway-egress-vpc-pattern/blob/master/README.md)[5](https://docs.aws.amazon.com/prescriptive-guidance/latest/integrate-third-party-services/architecture-3.html)

1. [https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/](https://www.cloudoptimo.com/blog/mastering-aws-transit-gateway-architecture-use-cases-and-best-practices/)
2. [https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
3. [https://aws.amazon.com/blogs/networking-and-content-delivery/aws-cloud-wan-and-aws-transit-gateway-migration-and-interoperability-patterns/](https://aws.amazon.com/blogs/networking-and-content-delivery/aws-cloud-wan-and-aws-transit-gateway-migration-and-interoperability-patterns/)
4. [https://github.com/aws-samples/aws-transit-gateway-egress-vpc-pattern/blob/master/README.md](https://github.com/aws-samples/aws-transit-gateway-egress-vpc-pattern/blob/master/README.md)
5. [https://docs.aws.amazon.com/prescriptive-guidance/latest/integrate-third-party-services/architecture-3.html](https://docs.aws.amazon.com/prescriptive-guidance/latest/integrate-third-party-services/architecture-3.html)
6. [https://docs.aws.amazon.com/solutions/latest/network-orchestration-aws-transit-gateway/architecture-diagram.html](https://docs.aws.amazon.com/solutions/latest/network-orchestration-aws-transit-gateway/architecture-diagram.html)
7. [https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
8. [https://d1.awsstatic.com/events/reinvent/2019/REPEAT_1_AWS_Transit_Gateway_reference_architectures_for_many_VPCs_NET406-R1.pdf](https://d1.awsstatic.com/events/reinvent/2019/REPEAT_1_AWS_Transit_Gateway_reference_architectures_for_many_VPCs_NET406-R1.pdf)
9. [https://docs.aviatrix.com/documentation/v7.2/network/tgw-design-patterns.html?expand=true](https://docs.aviatrix.com/documentation/v7.2/network/tgw-design-patterns.html?expand=true)
10. [https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/inspection-deployment-models-with-AWS-network-firewall-ra.pdf](https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/inspection-deployment-models-with-AWS-network-firewall-ra.pdf)