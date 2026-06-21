---
longform:
  format: single
  title: Kafka Client Keystore & Truststore Generation Guide
title: Kafka Client Keystore & Truststore Generation Guide
---


This document describes the end-to-end process for generating a Kafka client truststore and keystore using OpenSSL, an internal CA, and Java KeyStore (JKS) format.

---

# Prerequisite: Identity Matching

Since the Kafka cluster is configured with **mTLS authentication** and extracts the principal from the client certificate, the **Common Name (CN)** in the certificate must exactly match the Kafka username used by the application.

### Example

If the application authenticates as:

```text
app_producer_v3
```

Then the CSR must be generated with:

```text
CN=app_producer_v3
```

Failure to match the username and certificate CN will result in authentication failures.

---

# Step 1: Create the Client Truststore

The client application must trust the Kafka broker certificates.

Import the CA bundle into a Java truststore.

```bash
keytool -importcert \
  -alias CARoot \
  -keystore client.truststore.jks \
  -file /root/signed-cert/ca-bundle.crt \
  -storepass confluenttruststorepass \
  -noprompt
```

## Output

```text
client.truststore.jks
```

---

# Step 2: Generate Client Private Key and CSR

Generate a private key and Certificate Signing Request (CSR).

Replace `<CLIENT_USERNAME>` with the application username.

```bash
openssl req -new -newkey rsa:4096 -nodes \
  -keyout client.key \
  -out client.csr \
  -subj "/C=SA/ST=Riyadh/L=Riyadh/O=albtests/OU=IT/CN=<CLIENT_USERNAME>"
```

## Example

```bash
openssl req -new -newkey rsa:4096 -nodes \
  -keyout client.key \
  -out client.csr \
  -subj "/C=SA/ST=Riyadh/L=Riyadh/O=albtests/OU=IT/CN=app_producer_v3"
```

## Generated Files

```text
client.key
client.csr
```

### Note

Client certificates typically do **not require Subject Alternative Names (SANs)** because Kafka uses the certificate CN as the client identity.

---

# Step 3: Sign the CSR

Submit the generated CSR to the internal Certificate Authority (CA).

## Input

```text
client.csr
```

## Signed By

```text
Albtests Issuing CA
```

## Output

```text
client.crt
```

---

# Step 4: Create the Client Keystore

Since the private key was generated using OpenSSL, it must first be packaged into a PKCS12 file and then converted into a Java KeyStore (JKS).

---

## Step 4A: Create PKCS12 Bundle

Combine:

- Client private key
- Signed client certificate
- CA certificate chain

```bash
openssl pkcs12 -export \
  -in client.crt \
  -inkey client.key \
  -certfile /root/signed-cert/ca-bundle.crt \
  -out client.p12 \
  -name <CLIENT_USERNAME> \
  -passout pass:confluentkeystorestorepass
```

### Example

```bash
openssl pkcs12 -export \
  -in client.crt \
  -inkey client.key \
  -certfile /root/signed-cert/ca-bundle.crt \
  -out client.p12 \
  -name app_producer_v3 \
  -passout pass:confluentkeystorestorepass
```

## Output

```text
client.p12
```

---

## Step 4B: Convert PKCS12 to JKS

Convert the PKCS12 file into a Java KeyStore.

```bash
keytool -importkeystore \
  -srckeystore client.p12 \
  -srcstoretype PKCS12 \
  -srcstorepass confluentkeystorestorepass \
  -destkeystore client.keystore.jks \
  -deststorepass confluentkeystorestorepass \
  -destkeypass confluentkeystorestorepass \
  -alias <CLIENT_USERNAME> \
  -noprompt
```

### Example

```bash
keytool -importkeystore \
  -srckeystore client.p12 \
  -srcstoretype PKCS12 \
  -srcstorepass confluentkeystorestorepass \
  -destkeystore client.keystore.jks \
  -deststorepass confluentkeystorestorepass \
  -destkeypass confluentkeystorestorepass \
  -alias app_producer_v3 \
  -noprompt
```

## Output

```text
client.keystore.jks
```

---

# Verify Generated Artifacts

## Verify Truststore

```bash
keytool -list -v \
  -keystore client.truststore.jks \
  -storepass confluenttruststorepass
```

## Verify Keystore

```bash
keytool -list -v \
  -keystore client.keystore.jks \
  -storepass confluentkeystorestorepass
```

---

# Final Deliverables to Application Team

Provide the following files and passwords:

| File | Purpose |
|--------|----------|
| `client.truststore.jks` | Used by the client to validate Kafka broker certificates |
| `client.keystore.jks` | Used by the client for mTLS authentication |

## Required Passwords

### Truststore Password

```text
confluenttruststorepass
```

### Keystore Password

```text
confluentkeystorestorepass
```

---

# File Flow Overview

```text
Generate Key + CSR
        │
        ▼
   client.key
   client.csr
        │
        ▼
Submit CSR to CA
        │
        ▼
   client.crt
        │
        ▼
Combine:
- client.key
- client.crt
- ca-bundle.crt
        │
        ▼
   client.p12
        │
        ▼
Convert to JKS
        │
        ▼
client.keystore.jks

CA Bundle
    │
    ▼
client.truststore.jks
```

---

# Final Artifacts

```text
client.key                  (Private Key)
client.csr                  (Certificate Signing Request)
client.crt                  (Signed Certificate)
client.p12                  (PKCS12 Bundle)
client.keystore.jks         (Client Authentication)
client.truststore.jks       (Broker Trust Validation)
```