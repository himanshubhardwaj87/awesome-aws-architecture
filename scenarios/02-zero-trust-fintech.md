# Scenario 02: Zero-Trust Security Architecture for a Fintech Company

## 1. Problem Statement
A digital fintech company handles sensitive banking transactions, loan processing, and personal financial data. The architecture must adhere to regulatory compliance (PCI-DSS, SOC2) and enforce a strict **Zero-Trust Security Model**—where no user, device, or service is trusted by default, regardless of its location inside the network perimeter.

---

## 2. Requirements

### Functional
*   Enable secure customer login and multi-factor authentication (MFA).
*   Process bank account details and initiate money transfers securely.
*   Log all system administrator activities and access requests for compliance auditing.

### Non-Functional
*   **Security**: Implement strict Zero-Trust (Micro-segmentation, Identity-based authorization, Envelope Encryption).
*   **Auditability**: Complete audit trails of all API calls and modifications (100% compliance).
*   **Resiliency**: Protect endpoints from advanced cyber-attacks (Layer 7 DDoS, credential stuffing).

---

## 3. Architecture Diagram

![Zero-Trust Security Architecture](file:///Users/hbhardwaj/Code/awesome-aws-architecture/diagrams/zero_trust_fintech_architecture.png)

### Interactive Mermaid Blueprint
```mermaid
graph TD
    User[Fintech Client] -->|1. Authenticate Request| Cognito[Amazon Cognito User Pool]
    User -->|2. Scoped API Request with JWT JWT| CF[Amazon CloudFront CDN]
    CF --> WAF[AWS WAF]
    WAF --> ALB[Application Load Balancer]
    
    subgraph Secure_VPC_Boundary [VPC - Strict Micro-segmentation]
        subgraph Public_Subnets [Public Presentation Subnets]
            ALB
        end
        
        subgraph Private_App_Subnets [Private Compute Subnets]
            ALB -->|3. Route Traffic| ECS[ECS Fargate Microservices]
            ECS -->|4. Private API call| Endpoint[VPC Endpoint PrivateLink]
        end
        
        subgraph Private_Data_Subnets [Isolated Data Subnets]
            ECS -->|5. Access Customer Data| RDS[(Amazon Aurora PostgreSQL)]
        end
    end
    
    subgraph Shared_Security_Services [Shared Security Account]
        Endpoint -->|Private Backbone| KMS[AWS Key Management Service]
        Endpoint -->|Private Backbone| Secrets[AWS Secrets Manager]
    end
    
    GuardDuty[Amazon GuardDuty] -.->|Threat Intelligence Logging| SecurityHub[AWS Security Hub]
    SecurityHub -.->|Central Alerts| AuditTrail[(Log Archive S3 Bucket)]
```

---

## 4. Key AWS Services Used

| Service | Architectural Role | Scoped Purpose |
| :--- | :--- | :--- |
| **Amazon Cognito** | CIAM identity provider. | Handles customer registration, secure login, MFA, and JWT token issuance. |
| **AWS WAF** | Web Application Firewall. | Sanitizes HTTP requests, blocks bot attacks, and filters SQLi/XSS signatures. |
| **AWS PrivateLink** | Private network bridging. | Creates secure **Interface VPC Endpoints** to connect VPC services privately, bypassing the internet. |
| **AWS KMS** | Managed Encryption Keys. | Handles envelope encryption using Customer Managed Keys (CMKs) with auto-rotation. |
| **AWS Secrets Manager**| Vault database credentials. | Rotates and injects RDS database passwords securely into ECS Fargate containers at runtime. |
| **Amazon GuardDuty** | AI Threat Detection. | Monitors VPC Flow Logs, DNS queries, and CloudTrail logs for anomalies or intrusions. |
| **AWS Security Hub** | Compliance Dashboard. | Aggregates security alerts from GuardDuty, Inspector, and IAM Analyzer in a single view. |

---

## 5. Step-by-Step Design Walkthrough
1.  **Identity Verification**: The client app prompts users to log in via **Amazon Cognito**, which enforces multi-factor authentication (MFA) and returns a signed JSON Web Token (JWT).
2.  **API Entry Guard**: The client forwards the JWT in the Authorization header of HTTPS requests. Traffic routes through **Amazon CloudFront** protected by **AWS WAF** rules to block bots and suspicious request payloads.
3.  **Compute Authentication**: The **Application Load Balancer (ALB)** verifies the Cognito JWT signature at the boundary before forwarding the request to **Amazon ECS Fargate** microservices in private subnets.
4.  **Least-Privilege Services**: Each ECS Fargate container runs with a dedicated **Task IAM Role** that only allows actions required for its specific task. Container storage utilizes encrypted volumes.
5.  **Private Cloud Routing**: When the ECS service requires database credentials or decryption keys, it queries **AWS Secrets Manager** and **AWS KMS** privately via **Interface VPC Endpoints (AWS PrivateLink)**, preventing traffic from traversing the public internet.
6.  **Secure Storage**: Database tables inside the isolated private subnets are encrypted at rest using **KMS Customer Managed Keys (CMKs)**. Backups are automated and encrypted.
7.  **Continuous Threat Monitoring**: **Amazon GuardDuty** analyzes VPC flow logs and CloudTrail configurations in the background. If a threat is detected (e.g., an unauthorized API call), it triggers an alert in **AWS Security Hub** and forwards the audit trail to an immutable S3 bucket.

---

## 6. Design Patterns Applied
*   **Zero-Trust Network Access (ZTNA)**: No perimeter trust. Subnets are micro-segmented using Security Groups restricting ingress strictly to specific, verified source ports.
*   **Least Privilege Identity Federation**: Access to systems administrators is managed via **IAM Identity Center** using temporary, short-lived tokens and single sign-on (SSO).
*   **Envelope Encryption**: All sensitive financial data rows are encrypted locally within the application layer using data keys before writing to the database.

---

## 7. Trade-offs

### Pros
*   **Maximum Compliance & Protection**: Completely satisfies regulatory frameworks (PCI-DSS Level 1, SOC2, HIPAA) through pervasive encryption and micro-segmentation.
*   **Decoupled Secret Management**: Developers do not have access to database passwords or raw master encryption keys.
*   **Elimination of Internet Egress**: VPC endpoints route AWS service traffic internally, minimizing the risk of network snooping.

### Cons
*   **High Network Overhead Cost**: Deploying VPC Interface Endpoints across multiple subnets and AZs generates hourly endpoint fees and per-GB data processing charges.
*   **Key Lifecycle Risk**: Misconfiguring key rotation policies or losing access to KMS key policies can result in irreversible data loss.

---

## 8. When to Use This Pattern
*   Fintech, banking, and payment processing platforms.
*   Healthcare applications handling electronic health records (EHR).
*   Any enterprise application subject to strict compliance auditing.

---

## 9. Cost Estimate

*   **Total Monthly Cost**: ~$2,500 - $5,000 (higher due to security tooling).
*   **Key Cost Drivers**:
    *   *AWS PrivateLink Endpoints*: Billed per endpoint per hour per AZ (~$600 - $1,200/month for extensive VPC setups).
    *   *AWS KMS & Secrets Manager API Requests*: High throughput token evaluations.
    *   *Amazon GuardDuty & Security Hub*: Charges scale with volume of VPC logs analyzed.

---

## 10. Alternatives Considered & Why Rejected
*   **VPN Tunneling between Subnets**: Rejected. Traditional VPNs trust all traffic inside the tunnel. Zero-Trust requires continuous verification at each hop using application-level security and IAM policies.
*   **Host credentials in environment variables manually**: Rejected. Storing raw API keys or database passwords in environment variables exposes them to logging frameworks and developers, violating security compliance policies.

---

## 11. Failure Modes & Mitigations

### 1. KMS Key Policy Lockout
*   **Effect**: Applications lose access to decryption keys, shutting down the entire platform.
*   **Mitigation**: Restrict modification of KMS policies to a designated security master role, and enforce dual-authorization (multi-party approval) policies for key alterations.

### 2. GuardDuty Alerts Flood
*   **Effect**: Alert fatigue causes administrators to ignore critical intrusion warnings.
*   **Mitigation**: Configure **AWS Systems Manager Incident Manager** to automatically classify alerts, filtering out low-priority threats while escalating high-risk incidents immediately.

---

## 12. SA Interview Questions

### Question 1: How do you verify Cognito JWT tokens inside your backend microservices?
**Answer**: 
1.  Download the **JSON Web Key Set (JWKS)** public keys from the Cognito user pool endpoint.
2.  Validate the JWT signature locally using the public keys to verify the token was issued by your Cognito pool.
3.  Check the **Expiration time (`exp`)** claim to ensure the token is active.
4.  Verify the **Audience (`aud`)** claim matches your client application ID to prevent token substitution attacks.

### Question 2: Why do we use VPC Interface Endpoints instead of a standard NAT Gateway?
**Answer**: 
*   A **NAT Gateway** routes traffic from private subnets over the public internet to reach external AWS endpoints. This exposes traffic pathing to public routing layers and incurs high outbound data charges.
*   An **Interface VPC Endpoint (AWS PrivateLink)** provisions a private ENI inside your subnet. Traffic to AWS services is routed entirely within the private AWS network backbone, never touching the public internet. This enhances security and satisfies strict compliance regulations.

---

## 🔁 Interviewer Follow-Up Drills

Real interviews push past the first design. Practice defending it against these follow-ups.

### Follow-Up 1: Transaction volume grows 10x. What breaks first, and how do you fix it?
**Answer**: 
*   **First to break**: **AWS KMS request quotas**. Envelope encryption per financial row means `GenerateDataKey` or `Decrypt` calls on every read and write. KMS cryptographic request limits are shared per account per region, so all ECS services start getting throttled together.
*   **Fixes**:
    1.  Use the **AWS Encryption SDK data key caching** (bounded by maximum age and message count) so one KMS call covers many rows.
    2.  Cache database credentials in-process with the **Secrets Manager caching client** instead of calling `GetSecretValue` on every request.
    3.  Put **RDS Proxy** (with IAM authentication) in front of Aurora to absorb connection growth from the ECS Fargate services.
    4.  Request **Service Quotas** increases for KMS and the **Cognito** auth APIs ahead of time, and retune **WAF** rate-based rules so legitimate growth isn't blocked.

### Follow-Up 2: Cut the security platform bill by 40%. What do you change, and what do you give up?
**Answer**: 
*   **Interface endpoints** are the biggest line item ($600–$1,200/month). Centralize them in a shared-services VPC reached through **Transit Gateway**, and share them via **Route 53 Resolver** private hosted zones instead of duplicating endpoints in every VPC. *Give up*: Transit Gateway data-processing fees, and a larger blast radius if the central VPC is misconfigured.
*   Use free **Gateway Endpoints** for S3 (log archive and backups) instead of interface endpoints.
*   Reduce KMS request charges with **S3 Bucket Keys** on the log archive and with data key caching in the application.
*   Tune **GuardDuty** protection plans and **Security Hub** standards to the ones your auditors actually require, and shrink endpoint AZ counts in non-production only.
*   **Never cut**: Encryption, CloudTrail, MFA, or production AZ redundancy. They are PCI-DSS and SOC2 controls, not optional features.

### Follow-Up 3: You must move all data encryption to a new KMS key (e.g., a multi-Region key for DR) with zero downtime. How?
**Answer**: 
1.  Envelope encryption helps here: only the **encrypted data keys** need re-wrapping, not the financial data itself.
2.  Create the new **customer managed key**. Grant the ECS **Task IAM Roles** permission on both keys during the transition.
3.  Deploy the application to **encrypt new writes with the new key** and **decrypt with whichever key ID** is stored in the ciphertext metadata. This dual-read, single-write approach is backward compatible.
4.  Run a background job that calls `ReEncrypt` on stored data keys, which happens inside KMS so plaintext keys never leave the HSM. Throttle the job to stay under KMS quotas.
5.  After CloudTrail shows zero `Decrypt` calls on the old key, disable it (don't delete it). Keep it through the audit retention window.
*   For **Aurora storage encryption** (a different key), use RDS Blue/Green or a snapshot-copy-and-restore process with the new key.

### Follow-Up 4: The KMS interface VPC endpoint (or KMS itself) becomes unreachable. What's the blast radius, and how do you recover?
**Answer**: 
*   **Blast radius**: Every service that decrypts data fails closed. Account balances, transfers, and loan records become unreadable, and Aurora may be unable to reopen after a restart. This is effectively a platform-wide outage, which is the intended trade-off for zero trust.
*   **Mitigations**:
    *   Deploy the **PrivateLink endpoint in every AZ** with private DNS so ENI failure in one AZ is isolated.
    *   Short-lived **data key caches** keep in-flight traffic working for minutes during a brief blip.
    *   Guard **endpoint policies and security groups** with SCPs and AWS Config rules. Most real outages here are self-inflicted misconfigurations.
    *   Use **KMS multi-Region keys** so a DR region can decrypt replicated data independently.
*   **Detection**: Set CloudWatch alarms on KMS `ThrottlingException` and `AccessDenied` spikes, and route them to **Incident Manager** as severity-1 incidents.

### Follow-Up 5: A regulator now requires single-tenant, customer-controlled HSMs and proof that AWS admins can't access key material. What changes?
**Answer**: 
*   Back the KMS keys with a **KMS custom key store on AWS CloudHSM** (FIPS 140-3 Level 3, single-tenant). Applications keep calling the same KMS APIs, so ECS code doesn't change.
*   For keys held outside AWS entirely, use an **External Key Store (XKS)** backed by your own on-premises HSM.
*   Deploy CloudHSM as a **cluster of at least 2 HSMs across AZs**. You now own HSM availability, user management, and backups.
*   Add **Amazon Macie** to find stray PAN or PII in S3, and use **AWS Payment Cryptography** if card PIN/CVV operations are in scope.
*   **Trade-offs**: Significantly higher cost (each HSM is billed hourly), higher latency than native KMS, and a new failure domain to monitor.
