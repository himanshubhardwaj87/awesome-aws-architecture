# Security & Compliance Patterns on AWS

Security is the foundational pillar of the AWS Well-Architected Framework. AWS operates on a **Shared Responsibility Model**, where AWS secures the infrastructure ("of the cloud"), while the customer secures everything built on top ("in the cloud").

---

## 🏛️ Shared Responsibility Model

```
+--------------------------------------------------------------------------------+
|                        Customer Responsibility ("In the Cloud")                 |
|  - Customer Data     - IAM Access Controls     - Operating System Configuration|
|  - Network Security (Security Groups/NACLs)    - Encryption at rest/transit    |
+--------------------------------------------------------------------------------+
|                            AWS Responsibility ("Of the Cloud")                 |
|  - Physical Security of Data Centers           - Global Infrastructure (AZs)   |
|  - Hypervisor virtualization management         - Native Storage and DB Hardware|
+--------------------------------------------------------------------------------+
```

---

## 🔒 VPC Security Layer Architecture

The diagram below outlines the standard network security posture for a secure web workload, implementing public/private subnets, Security Groups, NACLs, VPC Endpoints, and AWS WAF.

```
                  [ Internet Traffic ]
                           │
                           ▼
                 [ AWS Shield / WAF ]
                           │
                           ▼
               [ Internet Gateway (IGW) ]
                           │
                           ▼
                [ Network ACL (NACL) ]  <-- Stateless Subnet Guard
                           │
                           ▼
      +────────────────────VPC Boundary────────────────────+
      |  [ Public Subnet ]                                 |
      |     └─► [ ALB Security Group ] (Stateful Port 443)  |
      |              │                                     |
      |              ▼                                     |
      |  [ Private Subnet ]                                |
      |     └─► [ App EC2 Security Group ]                 |
      |              │                                     |
      |              ▼                                     |
      |  [ Data Subnet ]                                   |
      |     └─► [ Aurora DB Security Group ]               |
      +────────────────────────────────────────────────────+
```

---

## Key AWS Security Services

### 1. Identity & Access Management (IAM)
*   **Principle of Least Privilege**: Grant only the minimal permissions required to complete a task.
*   **IAM Policies**: JSON documents defining permissions. Prefer Customer Managed Policies over AWS Managed Policies for tighter control.
*   **Service Control Policies (SCPs)**: Guardrails applied at the AWS Organizations level to restrict maximum permissions across multiple accounts.

### 2. AWS Key Management Service (KMS)
*   **Symmetric Encryption**: Uses a single key to encrypt and decrypt data. Integrated natively across AWS storage services.
*   **Envelope Encryption**: The practice of encrypting data with a Data Key, and encrypting the Data Key with a Root Customer Managed Key (CMK) in KMS. This optimizes performance when encrypting large volumes of data.

### 3. Edge Security: AWS WAF & Shield
*   **AWS Shield Standard**: Automatically enabled for all AWS accounts, protecting against common Layer 3 and 4 DDoS attacks.
*   **AWS Shield Advanced**: Offers active mitigation support and cost protection against application-layer DDoS surges.
*   **AWS WAF**: Web Application Firewall integrated with ALB, CloudFront, or API Gateway. Protects against SQL Injection, Cross-Site Scripting (XSS), and custom rate-limit attacks.

---

## Data Encryption: Transit vs. Rest

*   **Encryption in Transit**: Natively enforced using TLS/HTTPS. Offload TLS termination safely at the Application Load Balancer using certificates managed by **AWS Certificate Manager (ACM)**.
*   **Encryption at Rest**: Enforced using KMS keys. Ensure S3 buckets, RDS databases, EBS volumes, and DynamoDB tables use Customer Managed Keys (CMKs) to satisfy enterprise compliance requirements.

---

## Common Pitfalls in Security Designs
*   **Overly permissive Security Groups**: Allowing `0.0.0.0/0` ingress on database security groups. Access must be restricted strictly to target source security groups.
*   **Using Root User for daily operations**: Creating access keys for the master AWS root account. Daily administration must be managed via scoped IAM identity roles.
*   **Neglecting VPC Endpoints**: Routing database/S3 traffic from private subnets over the public internet via NAT Gateways. (Mitigation: Use **Gateway VPC Endpoints** for S3/DynamoDB to route traffic privately within the AWS backbone).

---

## SA Interview Questions on Security

### Question 1: What is the difference between Security Groups and Network Access Control Lists (NACLs)?
**Answer**: 
*   **Security Groups** are **stateful** firewall rules applied at the **elastic network interface (ENI)** level. Because they are stateful, return traffic is automatically allowed. They only support ALLOW rules and are evaluated collectively as a whole.
*   **NACLs** are **stateless** security guards applied at the **subnet boundary** level. Because they are stateless, you must configure both inbound and outbound rules explicitly. NACLs support both ALLOW and DENY rules and are evaluated in chronological order based on rule numbers.

### Question 2: Explain Envelope Encryption and how AWS uses it.
**Answer**: 
Directly sending large files to KMS to encrypt is inefficient and runs into network limits. AWS resolves this using **Envelope Encryption**:
1.  When a service (e.g., S3) wants to store an encrypted object, it requests a **Data Key** from KMS using the Master Key.
2.  KMS returns a plaintext Data Key and an encrypted version of that Data Key.
3.  The S3 service uses the plaintext Data Key to encrypt the file locally at high speed.
4.  The plaintext Data Key is deleted from S3 memory, and the encrypted file is stored alongside the encrypted Data Key.
5.  To decrypt, S3 sends the encrypted Data Key back to KMS. KMS decrypts it using the Master Key and returns the plaintext Data Key, which S3 uses to decrypt the file.

### Question 3: How do you protect a public-facing API on AWS from brute-force brute DDoS attacks and SQL injection exploits?
**Answer**: 
1.  Deploy **AWS WAF** on the API Gateway or CloudFront distribution. Configure rules to inspect payloads for standard SQL injection signatures and sanitize inputs.
2.  Configure a **Rate-Limiting Rule** in WAF to limit client requests (e.g., block client IPs that exceed 100 requests per 5-minute period).
3.  Enable **AWS Shield Advanced** on the public-facing distribution to automatically mitigate infrastructure-level DDoS attacks.

### Question 4: How do you design end-to-end security using the 'defense in depth' model for a serverless microservice?
**Answer**: 
Defense in Depth means implementing layered security controls across the entire infrastructure stack to ensure that a breach at one layer is contained (reducing the blast radius). For a serverless architecture (e.g., API Gateway -> Lambda -> DynamoDB):

1.  **Identity Layer (Authentication & Authorization)**: Ensure only authenticated requests reach the API Gateway by using **Amazon Cognito** User Pools or custom Lambda Authorizers.
2.  **Network Layer (Data in Transit)**: Enforce HTTPS using SSL/TLS certificates managed in **AWS Certificate Manager (ACM)**. Deploy API Gateway inside a private subnet and use VPC Endpoints to restrict access.
3.  **Edge Protection Layer**: Attach **AWS WAF** to API Gateway to protect endpoints from common OWASP Top 10 vulnerabilities (like SQL injection or Cross-Site Scripting) and configure rate limits to prevent brute-force or DDoS exploits.
4.  **Compute/Runtime Layer**: Scan Lambda function packages using **Amazon Inspector** to identify runtime vulnerabilities and outdated dependencies. Use highly-scoped IAM Execution Roles applying the **Principle of Least Privilege**, and configure resource policies to only allow API Gateway to invoke the Lambda.
5.  **Secrets Management Layer**: Never hardcode database credentials or API keys. Store them in **AWS Secrets Manager** and configure automatic credential rotation policies.
6.  **Data at Rest Layer**: Enable **AWS KMS Envelope Encryption** with Customer Managed Keys (CMKs) to encrypt DynamoDB tables and CloudWatch log groups.

