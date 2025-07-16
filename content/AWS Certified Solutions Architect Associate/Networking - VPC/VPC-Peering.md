---
longform:
  format: single
  title: VPC Peering
title: VPC-Peering
---
### **VPC Peering Explained with Diagram**  
**VPC Peering** allows two Virtual Private Clouds (VPCs) to communicate with each other privately using AWS's internal network, without traversing the public internet. It enables secure, low-latency connections between VPCs in the same or different AWS accounts/regions.

---

### **Key Features of VPC Peering**  
1. **Private Connectivity** – Traffic stays within AWS’s network.  
2. **No Single Point of Failure** – Unlike VPN, it doesn’t rely on a gateway.  
3. **Cross-Account & Cross-Region Support** – Can connect VPCs in different AWS accounts or regions.  
4. **No Transitive Peering** – If VPC A is peered with VPC B, and VPC B is peered with VPC C, **VPC A cannot talk to VPC C** directly.  
5. **CIDR Block Restrictions** – Peered VPCs must have non-overlapping IP ranges.  

---

### **VPC Peering Diagram**  
Here’s a simple illustration of VPC Peering between two VPCs:

```
+---------------------+          +---------------------+
|      VPC A          |          |      VPC B          |
|  (10.0.0.0/16)      | <------> |  (192.168.0.0/16)   |
|                     |          |                     |
| +----------------+  |          |  +----------------+ |
| |   Subnet 1     |  |          |  |   Subnet 1     | |
| | (10.0.1.0/24)  |  |          |  | (192.168.1.0/24)| |
| |   EC2 Instance |  |          |  |   EC2 Instance | |
| +----------------+  |          |  +----------------+ |
+---------------------+          +---------------------+
          ▲                                  ▲
          |                                  |
          ▼                                  ▼
+---------------------+          +---------------------+
|   Route Table A     |          |   Route Table B     |
| Destination: 192.168.0.0/16 |  | Destination: 10.0.0.0/16   |
| Target: pcx-123 (Peering) |  | Target: pcx-123 (Peering)  |
+---------------------+          +---------------------+
```

---

### **How VPC Peering Works**  
1. **Establish a Peering Connection**  
   - One VPC owner sends a peering request to another VPC (same/different account).  
   - The other VPC owner must accept the request.  

2. **Update Route Tables**  
   - Both VPCs must add routes pointing to each other’s CIDR block via the peering connection (`pcx-xxx`).  

3. **Optional: Update Security Groups/NACLs**  
   - Ensure security groups allow traffic from the peered VPC’s IP range.  

---

### **Use Cases of VPC Peering**  
✔ **Multi-tier applications** – Connect frontend and backend VPCs.  
✔ **Shared services** – Allow multiple VPCs to access a central database.  
✔ **Cross-account access** – Securely link VPCs owned by different teams.  

---

### **Limitations**  
❌ **No transitive peering** (must set up direct connections).  
❌ **Overlapping CIDR blocks are not allowed**.  
❌ **Region constraints** – Inter-region peering may have higher latency.  
