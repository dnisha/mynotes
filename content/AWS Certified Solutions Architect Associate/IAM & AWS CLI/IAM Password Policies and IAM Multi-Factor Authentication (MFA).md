---
longform:
  format: single
  title: IAM Password Policies and IAM Multi-Factor Authentication (MFA)
title: IAM Password Policies and IAM Multi-Factor Authentication (MFA)
---

---

### **1. IAM Password Policy**
- **Definition**: A set of rules that enforce security requirements for IAM user passwords.  
- **Purpose**: Prevent weak passwords and improve account security.  

#### **Key Configurable Settings**  
1. **Minimum Password Length** (e.g., 12 characters).  
2. **Password Complexity**: Require:  
   - Uppercase letters (A-Z).  
   - Lowercase letters (a-z).  
   - Numbers (0-9).  
   - Special characters (`!@#$%^&*`).  
3. **Password Rotation**:  
   - Enable **password expiration** (e.g., 90 days).  
   - Prevent **password reuse** (e.g., last 5 passwords).  
4. **Account Lockout**:  
   - Lock accounts after failed login attempts (not natively supported; requires AWS Organizations or custom solutions).  

#### **How to Set Up**  
- **AWS Console**:  
  - Navigate to **IAM → Account Settings → Password Policy**.  
- **AWS CLI**:  

  ```bash
  aws iam update-account-password-policy \
    --minimum-password-length 12 \
    --require-symbols \
    --require-numbers \
    --require-uppercase-characters \
    --require-lowercase-characters \
    --allow-users-to-change-password \
    --max-password-age 90 \
    --password-reuse-prevention 5
  ```

#### **Best Practices**  
- Enforce **at least 12-character passwords** with complexity.  
- Enable **password expiration** (e.g., 90 days).  
- Allow users to **change their own passwords**.  

---

### **2. IAM Multi-Factor Authentication (MFA)**
- **Definition**: A security mechanism requiring users to provide **two forms of authentication**:  
  1. Something they **know** (password).  
  2. Something they **have** (MFA device).  
- **Purpose**: Protect against compromised credentials (e.g., stolen passwords).  

#### **MFA Device Options**  
1. **Virtual MFA Devices** (e.g., Google Authenticator, Microsoft Authenticator, Authy).  
2. **Hardware MFA Devices** (e.g., YubiKey, Gemalto).  
3. **FIDO Security Keys** (WebAuthn standard, e.g., YubiKey).  
4. **SMS-based MFA** (Less secure; deprecated for root accounts).  

#### **How to Enable MFA**  
- **For an IAM User**:  
  1. Go to **IAM → Users → Security Credentials → MFA**.  
  2. Choose a device type and follow setup instructions.  
- **For the Root Account**:  
  - **Critical** to enable MFA to prevent full account compromise.  

#### **AWS CLI Example**  
```bash
# List MFA devices for a user
aws iam list-mfa-devices --user-name john
```

#### **Best Practices**  
- **Enforce MFA** for:  
  - **Root account** (highest priority).  
  - **Privileged IAM users** (e.g., admins).  
  - **API access** (via temporary credentials with MFA).  
- Use **hardware/U2F keys** for high-security needs.  
- Avoid **SMS-based MFA** (phishing risk).  

---