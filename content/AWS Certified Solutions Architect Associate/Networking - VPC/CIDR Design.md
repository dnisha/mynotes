---
longform:
  format: single
  title: CIDR Design
title: CIDR Design
---

### **Subnet CIDR Allocation Strategy**
Using `10.0.0.0/16` VPC with /24 subnet masks (256 IPs each)

| Subnet | CIDR Block | Type | AZ | Purpose |
|--------|------------|------|----|---------|
| Public Subnet 1 | `10.0.1.0/24` | Public | us-east-1a | Public-facing resources |
| Public Subnet 2 | `10.0.2.0/24` | Public | us-east-1b | Public-facing resources |
| Private Subnet 1 | `10.0.10.0/24` | Private | us-east-1a | Application tier |
| Private Subnet 2 | `10.0.11.0/24` | Private | us-east-1b | Application tier |
| Private Subnet 3 | `10.0.20.0/24` | Private | us-east-1a | Data tier (RDS) |
| Private Subnet 4 | `10.0.21.0/24` | Private | us-east-1b | Data tier (RDS) |

### **Alternative CIDR Design (More Scalable)**

If you need more room for growth, consider this design:

```hcl
# Alternative: Using /20 subnets (4096 IPs each) for larger workloads
VPC: 10.0.0.0/16

Public Subnets:
- 10.0.0.0/20   (10.0.0.0 - 10.0.15.255)   # AZ A
- 10.0.16.0/20  (10.0.16.0 - 10.0.31.255)  # AZ B

Private Subnets:
# Application Layer
- 10.0.32.0/20  (10.0.32.0 - 10.0.47.255)  # AZ A
- 10.0.48.0/20  (10.0.48.0 - 10.0.63.255)  # AZ B

# Data Layer
- 10.0.64.0/20  (10.0.64.0 - 10.0.79.255)  # AZ A
- 10.0.80.0/20  (10.0.80.0 - 10.0.95.255)  # AZ B
```
