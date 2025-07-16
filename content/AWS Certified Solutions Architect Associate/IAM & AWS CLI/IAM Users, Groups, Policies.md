---
longform:
  format: single
  title: IAM Users, Groups, Policies
title: IAM Users, Groups, Policies
---
![[IAM Users, Groups, Policies.png]]

---

### **1. IAM Users**
- **Definition**: An IAM User represents a person or service that interacts with AWS.  
- **Purpose**: Each user has unique credentials (username/password or access keys) to access AWS resources.  
- **Use Case**: Assign individual permissions to users (e.g., developers, admins).  
- **Key Features**:  
  - Can belong to multiple **Groups**.  
  - Supports **Multi-Factor Authentication (MFA)**.  
  - Can have **access keys** for programmatic AWS API/CLI access.  

---

### **2. IAM Groups**
- **Definition**: A collection of IAM Users.  
- **Purpose**: Simplify permission management by assigning permissions to a group instead of individual users.  
- **Use Case**: Group users by job function (e.g., `Admins`, `Developers`, `Auditors`).  
- **Key Features**:  
  - A user can belong to multiple groups.  
  - Groups **cannot be nested** (no subgroups).  
  - No default group (users don’t have to belong to one).  

---

### **3. IAM Policies**
- **Definition**: JSON documents that define permissions (allow/deny access to AWS resources).  
- **Purpose**: Grant fine-grained control over who can do what in AWS.  
- **Types**:  
  - **Managed Policies**:  
    - AWS pre-defined (e.g., `AmazonS3FullAccess`) or custom-created.  
    - Can be attached to multiple Users/Groups/Roles.  
  - **Inline Policies**:  
    - Embedded directly into a single User/Group/Role.  
    - Deleted when the parent entity is deleted.  
- **Policy Structure**:  

  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::example-bucket/*"
      }
    ]
  }
  ```
  
  - **Effect**: `Allow` or `Deny`.  
  - **Action**: API operations (e.g., `ec2:StartInstance`).  
  - **Resource**: AWS service ARNs (e.g., an S3 bucket ARN).  

---

### **Key Relationships**
1. **Users → Groups**: Users inherit permissions from groups they belong to.  
2. **Users/Groups → Policies**: Permissions are assigned via policies (managed or inline).  
3. **Principle of Least Privilege**: Grant only the permissions necessary for a task.  

---

### **Best Practices**
- Use **Groups** to manage permissions at scale.  
- Prefer **Managed Policies** over inline policies for reusability.  
- Regularly **audit permissions** using IAM Access Analyzer.  
- Enable **MFA** for sensitive operations.  

---

### **Example Workflow**
1. Create an IAM User (`john`).  
2. Add `john` to the `Developers` group.  
3. Attach a managed policy (e.g., `AmazonEC2ReadOnlyAccess`) to the group.  
4. Result: `john` can now describe EC2 instances but cannot modify them.  

---