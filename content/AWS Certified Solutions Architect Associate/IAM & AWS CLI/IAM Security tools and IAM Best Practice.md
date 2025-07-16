---
longform:
  format: single
  title: IAM Security tools and IAM Best Practice
title: IAM Security tools and IAM Best Practice
---
# **AWS IAM: Roles, Security Tools & Best Practices**  

## **1. IAM Roles for AWS Services**  
IAM roles allow AWS services to securely interact with other AWS resources without hardcoding credentials.  

### **Key Features:**  
- **Temporary Credentials:** Roles provide short-term credentials instead of long-term access keys.  
- **Service-Linked Roles:** Predefined by AWS for services like EC2, Lambda, RDS, etc.  
- **Cross-Account Access:** Enables secure access between different AWS accounts.  

### **Common Use Cases:**  
- EC2 instances needing access to S3.  
- Lambda functions interacting with DynamoDB.  
- AWS Step Functions calling other AWS services.  

### **Example:**  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## **2. IAM Security Tools**  

### **A. IAM Credentials Report**  
Generates a detailed report of all IAM users and their credentials (passwords, access keys, MFA status).  

**How to Generate:**  
```bash
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 --decode > report.csv
```

**Key Checks:**  
✔ Expired passwords/keys  
✔ Unused access keys (>90 days old)  
✔ Users without MFA  

### **B. IAM Access Advisor**  
Shows service permissions granted to a user/role and when they were last accessed.  

**How to Use:**  
- Navigate to **IAM → Users/Roles → Access Advisor**.  
- Identifies unused permissions for cleanup (least privilege principle).  

---

## **3. IAM Best Practices**  

### **A. Least Privilege Principle**  
- Grant only necessary permissions.  
- Regularly review policies using **Access Advisor**.  

### **B. Enable MFA (Multi-Factor Authentication)**  
- Required for root and privileged users.  

### **C. Use IAM Roles Instead of Access Keys**  
- Avoid long-term access keys; prefer roles for AWS services.  

### **D. Rotate Credentials Regularly**  
- Enforce password & access key rotation policies.  

### **E. Monitor with AWS CloudTrail & AWS Config**  
- Track IAM changes and compliance.  

### **F. Avoid Root Account Usage**  
- Use IAM users/roles for daily operations.  

### **G. Use Policy Conditions**  
- Restrict access by IP, time, or MFA.  
```json
"Condition": {
  "IpAddress": {"aws:SourceIp": ["192.0.2.0/24"]},
  "Bool": {"aws:MultiFactorAuthPresent": "true"}
}
```

---

## **Conclusion**  
- **IAM Roles** provide secure, temporary access for AWS services.  
- **Credentials Report & Access Advisor** help audit and refine permissions.  
- **Best Practices** ensure security, compliance, and least privilege access.  