---
longform:
  format: single
  title: Main Route Table And Subnet Specific Route Table
title: Main Route Table And Subnet Specific Route Table
---
### **Difference Between Main Route Table (Main RT) and Subnet-Specific Route Table (Custom RT) in AWS VPC**

In AWS VPC, **route tables (RTs)** control how traffic is routed within the VPC and to external networks. There are two types of route tables:

1. **Main Route Table (Default Route Table)**  
   - Created automatically when a VPC is created.  
   - Controls routing for **all subnets that do not have an explicit (custom) route table associated**.  
   - Can be modified but **cannot be deleted**.  

2. **Subnet-Specific (Custom) Route Table**  
   - Manually created by users.  
   - Explicitly associated with one or more subnets to override the main route table.  
   - Allows **fine-grained control** over routing for specific subnets.  

---

### **Key Differences**  

| Feature                     | **Main Route Table**                          | **Subnet-Specific Route Table**               |
|-----------------------------|-----------------------------------------------|-----------------------------------------------|
| **Creation**                | Automatically created with VPC.               | Manually created by the user.                 |
| **Association**             | Applies to subnets **not explicitly linked** to a custom RT. | Must be **explicitly assigned** to a subnet. |
| **Modification**            | Can be edited but **cannot be deleted**.      | Can be edited, replaced, or deleted.         |
| **Default Routes**          | Initially has only a local route (`10.0.0.0/16 → local`). | Starts empty (only local route if not modified). |
| **Use Case**                | Best for subnets with **default routing** (e.g., private subnets). | Used for **custom routing** (e.g., public subnets with IGW, VPN, or peering). |

---

### **Example Scenario with Diagram**  

#### **VPC Setup:**
- **VPC CIDR:** `10.0.0.0/16`  
- **Subnets:**  
  - **Public Subnet:** `10.0.1.0/24` (needs internet access via IGW)  
  - **Private Subnet:** `10.0.2.0/24` (no internet access)  

#### **Route Tables:**
1. **Main Route Table (Default)**  
   - Routes:  
     - `10.0.0.0/16 → local` (default)  
   - Associated with **Private Subnet** (since no custom RT is assigned).  

2. **Custom Route Table (Public-RT)**  
   - Routes:  
     - `10.0.0.0/16 → local`  
     - `0.0.0.0/0 → igw-123` (Internet Gateway)  
   - Explicitly **associated with Public Subnet**.  

```
+---------------------VPC (10.0.0.0/16)---------------------+
|                                                           |
|  +------------------+        +------------------+        |
|  |   Public Subnet  |        |  Private Subnet  |        |
|  | (10.0.1.0/24)    |        | (10.0.2.0/24)    |        |
|  |                  |        |                  |        |
|  | EC2 Instance     |        | EC2 Instance     |        |
|  +--------▲---------+        +--------▲---------+        |
|           |                            |                 |
|  +--------▼---------+        +--------▼---------+        |
|  | Custom RT (Public)|        | Main RT (Default)|        |
|  | Routes:           |        | Routes:          |        |
|  | 10.0.0.0/16 → local|       | 10.0.0.0/16 → local|     |
|  | 0.0.0.0/0 → igw-123|       +------------------+        |
|  +------------------+                                    |
+----------------------------------------------------------+
```

---

### **When to Use Which?**  
✅ **Use Main Route Table:**  
   - For subnets that **don’t need special routing** (e.g., private DB subnets).  

✅ **Use Custom Route Table:**  
   - For **public subnets** (needs `0.0.0.0/0 → IGW`).  
   - For **VPN/Peering/NAT Gateways** (requires custom routes).  
   - For **isolated subnets** (blocking internet access).  

---

### **How to Check & Modify in AWS Console?**  
1. Go to **VPC Dashboard → Route Tables**.  
2. The **Main RT** is marked as **"Yes"** under **"Main"** column.  
3. To assign a **Custom RT** to a subnet:  
   - Select the RT → **Subnet Associations → Edit → Choose Subnet**.  

---

### **Key Takeaway**  
- **Main RT** = Default routing (applies if no custom RT is assigned).  
- **Subnet-Specific RT** = Custom routing (overrides Main RT for selected subnets).  