# Scenario 05: SaaS Multi-Tenant Architecture on AWS

## 1. Problem Statement
A software-as-a-service (SaaS) provider is building a multi-tenant enterprise application. The platform must isolate customer data securely (mitigating "noisy neighbor" issues and preventing cross-tenant data leaks), scale cost-effectively, and automate onboarding for new tenants dynamically.

---

## 2. Requirements

### Functional
*   Automate tenant onboarding and environment provisioning dynamically.
*   Enforce strict isolation policies for tenant databases and compute environments.
*   Track and attribute AWS usage costs accurately per tenant for accurate billing.

### Non-Functional
*   **Tenant Security**: Prevent cross-tenant data access (100% data isolation guarantee).
*   **Scale**: Support hundreds of tenants without hitches, handling high transaction loads.
*   **Operational Excellence**: Manage all tenant configurations and environments from a central console.

---

## 3. Architecture Diagram

This architecture illustrates a **Bridge Isolation Model**, combining **Silo Isolation** (dedicated, isolated resources for high-tier enterprise tenants) and **Pool Isolation** (shared compute resources with logical IAM-based data isolation for standard-tier tenants).

![SaaS Multi-Tenant with Silo and Pool Isolation Architecture](file:///Users/hbhardwaj/Code/awesome-aws-architecture/diagrams/saas_multi_tenant_architecture.png)

### Interactive Mermaid Blueprint
```mermaid
graph TD
    Client[Tenant Client Applications] -->|Tenant Header Context| ALB[Application Load Balancer]
    ALB --> WAF[AWS WAF]
    WAF --> Router[Tenant Traffic Routing Lambda]
    
    subgraph Multi_Tenant_Bridge_Model [Bridge Tenant Isolation Architecture]
        subgraph Tenant_Pool_Standard [Standard Tier - Pool Isolation]
            Router -->|logical isolation| ECS_Pool[ECS Shared Fargate Service]
            ECS_Pool -->|Dynamic Session Credentials| STS[AWS Security Token Service]
            ECS_Pool -->|Query with Tenant Tag| DynamoDB_Pool[(Shared Tenant DynamoDB Table)]
        end
        
        subgraph Tenant_Silo_Enterprise_A [Premium Tier Tenant A - Silo Isolation]
            Router -->|physical isolation| Account_A[Dedicated AWS Tenant Account A]
            Account_A --> ECS_A[ECS Dedicated Fargate Container]
            ECS_A --> RDS_A[(Dedicated Amazon Aurora DB Instance)]
        end
    end
    
    subgraph Control_Plane [Shared Core Admin Services]
        Onboarding[Admin Tenant Onboarding Engine] -->|Provision Accounts| ControlTower[AWS Control Tower]
        ControlTower -->|Deploy guardrails| SCPs[Service Control Policies]
    end
```

---

## 4. Key AWS Services Used

| Service | Architectural Role | Scoped Purpose |
| :--- | :--- | :--- |
| **AWS Control Tower** | Tenant Provisioning. | Automates the creation of isolated tenant accounts ("Silo") via account factories. |
| **AWS Organizations** | Account Management. | Consolidates billing and applies security policies across all tenant accounts. |
| **AWS STS** | Temporary Credentials. | Generates dynamic, short-lived IAM credentials containing tenant-context tags. |
| **Amazon DynamoDB** | Shared Pooled Database. | Stores pooled tenant data, enforcing row-level security using tenant partitioning. |
| **Amazon Aurora** | Dedicated Silo Database. | Stores data for high-tier enterprise tenants in isolated databases. |
| **Service Control Policies (SCPs)**| Core Security Guardrails.| Prevents tenant accounts from modifying logging, network, or security settings. |

---

## 5. Step-by-Step Design Walkthrough
### Phase A: Tenant Onboarding & Provisioning
1.  **Onboarding Trigger**: A new customer signs up for the SaaS service. The **Admin Tenant Onboarding Engine** receives the registration payload.
2.  **Tier Routing**:
    *   **Premium Tier (Silo)**: The onboarding engine calls the **AWS Control Tower Account Factory** API to provision a dedicated, isolated AWS account for the tenant. AWS Control Tower applies default OUs, security guardrails, and VPC configurations.
    *   **Standard Tier (Pool)**: The onboarding engine registers the new customer metadata inside a shared **DynamoDB Table**, assigning them a unique Tenant ID tag.

### Phase B: Query Routing & Isolation Enforcement
1.  **Client Egress**: Customer requests hit the **Application Load Balancer (ALB)**, which evaluates the tenant context header (e.g., `Tenant-ID: TenantA`).
2.  **Premium Route**: If the request belongs to a Premium Silo tenant, the **Traffic Routing Lambda** routes the request directly to the dedicated AWS account. Data is saved in a dedicated **Amazon Aurora DB** instance.
3.  **Standard Route**: If the request belongs to a Standard Pool tenant:
    *   The request routes to the shared **ECS Fargate** cluster.
    *   The ECS task queries the **AWS Security Token Service (STS)** to assume a dynamic IAM role containing session context attributes (e.g., `TenantID = TenantStandard`).
    *   STS returns temporary credentials. The application uses these keys to read/write data to a shared DynamoDB table. The DynamoDB IAM policy restricts query access strictly to rows matching the dynamic `TenantID` attribute (Row-level security).

---

## 6. Design Patterns Applied
*   **Silo Isolation Model**: Placing a tenant's entire compute and database stack inside a dedicated, physically isolated AWS account.
*   **Pool Isolation Model**: Tenants share the same compute resources and database instances. Isolation is enforced logically using IAM policies and application routing rules.
*   **Bridge Isolation Model**: Combining Silo and Pool architectures within the same platform to support different customer tiers.

---

## 7. Trade-offs

### Pros
*   **Tiered Pricing Capabilities**: Allows selling lower-cost logical tiers to standard customers, and premium, highly isolated physical tiers to enterprise clients.
*   **Zero Data Leakage (Silo)**: Physical account separation completely eliminates the risk of cross-tenant database access.
*   **High Cost-Efficiency (Pool)**: Shared resources minimize idle capacity costs, maximizing overall resource utilization.

### Cons
*   **High Complexity**: Managing two separate deployment and isolation architectures simultaneously requires significant engineering effort.
*   **Account Limit Barriers**: Running hundreds of dedicated silo accounts can hit default AWS Organization account limits.

---

## 8. When to Use This Pattern
*   B2B SaaS platforms selling services to diverse customer tiers (e.g., small business vs. massive enterprise).
*   Applications with strict regulatory compliance guidelines requiring complete physical data isolation for enterprise clients.

---

## 9. Cost Estimate

*   **Total Monthly Cost**: Highly variable (scales with tenant count).
*   **Key Cost Drivers**:
    *   *Silo Accounts*: Base costs multiply per silo account (e.g., running VPCs, ALBs, NAT Gateways per tenant).
    *   *Pooled Accounts*: Highly cost-effective (shared DynamoDB scales based on active requests).

---

## 10. Alternatives Considered & Why Rejected
*   **Silo-only Model**: Rejected. Running dedicated AWS accounts for small, low-paying standard customers is cost-prohibitive due to duplicate resource overhead (e.g., NAT Gateways, Load Balancers).
*   **Pool-only Model**: Rejected. Large enterprise clients often refuse to share databases with other tenants due to strict security guidelines and compliance regulations.

---

## 11. Failure Modes & Mitigations

### 1. Noisy Neighbor Performance Drops
*   **Effect**: A single pooled standard tenant runs massive, resource-heavy reports, slowing down performance for all other standard tenants.
*   **Mitigation**: Implement strict **API rate-limiting (WAF)** and resource throttling policies per tenant context.

### 2. Tenant Context Injection
*   **Effect**: A malicious tenant alters their header context, spoofing another tenant's identity to steal private data.
*   **Mitigation**: Enforce cryptographically signed JWT tokens for all tenant authentication. The routing layers must verify the signature before resolving tenant contexts.

---

## 12. SA Interview Questions

### Question 1: How do you enforce data isolation in a shared DynamoDB table (Pool model)?
**Answer**: 
1.  Design the DynamoDB table using `TenantID` as the Partition Key (PK).
2.  Configure your application to query **AWS Security Token Service (STS)** when handling tenant requests, requesting temporary credentials.
3.  Assign an **IAM Policy** to the assumed role that uses **IAM Policy Conditions** containing a dynamic policy variable:
    ```json
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:*:table/TenantSharedTable",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${aws:PrincipalTag/TenantID}"]
        }
      }
    }
    ```
4.  DynamoDB automatically rejects any requests attempting to read partitions matching other Tenant IDs, securing your data layer.

### Question 2: Why is AWS Control Tower preferred over basic CloudFormation for provisioning Silo tenant environments?
**Answer**: 
*   **CloudFormation** is a template engine that provisions resources within an existing AWS account. It does not create accounts, set up organizational hierarchies, or configure external security guardrails automatically.
*   **AWS Control Tower** operates at the Organizations level. Its **Account Factory** automates account creation, applies default security guardrails (Service Control Policies), establishes network paths, and registers accounts to centralized logging services in a single step, ensuring consistent governance.

---

## 🔁 Interviewer Follow-Up Drills

Real interviews push past the first design. Practice defending it against these follow-ups.

### Follow-Up 1: The tenant count grows 10x (hundreds to thousands). What breaks first, and how do you fix it?
**Answer**: 
*   **First to break**: Per-request overhead and hot keys in the **pool** tier.
    *   The **Tenant Traffic Routing Lambda** is in the hot path for every tenant.
    *   Calling **STS AssumeRole** on every request runs into STS rate limits.
    *   A large pool tenant becomes a **hot partition** on its `TenantID` key in the shared DynamoDB table.
*   **Fixes**:
    1.  Cache **STS session credentials per tenant** for their lifetime instead of per request.
    2.  **Write-shard** big tenants (`TenantID#<n>`) and update the IAM condition to `ForAllValues:StringLike` on `dynamodb:LeadingKeys` with `"${aws:PrincipalTag/TenantID}#*"`. The key is multivalued, so it needs the `ForAllValues` set operator, like the base policy above.
    3.  Move routing into **ALB listener rules** or an **API Gateway Lambda authorizer with caching** so tenant resolution isn't a per-request invocation.
    4.  **Silo** side: Queue account vending through **Control Tower Account Factory for Terraform (AFT)** and raise the **Organizations account quota** ahead of time.

### Follow-Up 2: Cut the platform cost by 40%. What do you change, and what do you give up?
**Answer**: 
*   **Silo overhead** dominates, because every premium account duplicates its NAT Gateways, ALB, and VPC.
    *   Centralize egress through **Transit Gateway**, or use **VPC sharing (AWS RAM)**, so silo accounts don't each run NAT Gateways.
    *   Move low-utilization silo databases to **Aurora Serverless v2**.
    *   *Give up*: A shared network plane weakens the "fully independent stack" story you sell to enterprise customers.
*   **Re-tier**: Move mid-size customers who don't need physical isolation to the pool, or to a **"pod" bridge model** (a few tenants per shared stack).
*   **Pool**: Switch the DynamoDB table from on-demand to **provisioned with auto-scaling** for steady load, and cover ECS Fargate with **Savings Plans** (with **Fargate Spot** for async jobs).
*   Enforce **per-tenant cost allocation tags** so you can see which tenants are unprofitable before cutting.

### Follow-Up 3: A Standard (pool) tenant upgrades to Premium (silo). How do you migrate them with zero downtime?
**Answer**: 
1.  Provision the tenant's dedicated account through **Control Tower Account Factory**, and deploy the silo stack (ECS Fargate and **Aurora**).
2.  **Backfill**: Query the tenant's partition (`TenantID` PK) from the shared DynamoDB table and transform it into the relational schema in Aurora.
3.  **CDC**: Stream ongoing changes with **DynamoDB Streams**. A Lambda with an **event filter on `TenantID`** applies them to Aurora until lag is near zero.
4.  **Validate**: Run shadow reads against both stores and reconcile row counts and checksums.
5.  **Cutover**: Flip the tenant's entry in the tenant registry so the **Routing Lambda** sends traffic to the silo account. Use a per-tenant write pause of seconds only if needed. Blast radius is one tenant.
6.  Keep the pool rows read-only for a rollback window, then delete them.

### Follow-Up 4: The Tenant Traffic Routing Lambda (or its tenant registry) fails. What's the blast radius, and how do you recover?
**Answer**: 
*   **Blast radius**: This is the highest-impact component because **every tenant in both tiers** routes through it. Unlike a single silo account failure (one tenant) or a DynamoDB pool issue (pool tenants only), a bad routing deploy is a platform-wide outage. It can also be a cross-tenant **misrouting** incident, which is worse than downtime.
*   **Prevention**:
    *   Deploy the routing Lambda with **CodeDeploy canary traffic shifting** on aliases and auto-rollback on error alarms.
    *   Set **reserved concurrency**, and cache the tenant-to-destination map in memory with a TTL.
*   **Fail closed**: If tenant context can't be verified from the **signed JWT**, reject the request. Never default-route it.
*   **Recovery**: Roll back the Lambda alias, and restore the tenant registry from **DynamoDB point-in-time recovery**. Long term, push static routing into ALB rules so the Lambda leaves the critical path.

### Follow-Up 5: A new enterprise customer requires that all their data stays in the EU (data residency). How does the bridge model handle it?
**Answer**: 
*   **Silo tenant**: Vend their account from **Control Tower** into an EU-only OU with an **SCP denying `aws:RequestedRegion`** outside approved EU regions. Deploy the silo stack and Aurora in `eu-west-1` with a tenant-specific **KMS key**.
*   **Pool tenants in the EU**: Stand up a **regional pool deployment** (ECS and a separate DynamoDB table in the EU). Do **not** use DynamoDB Global Tables, because it replicates every item to every replica region and would break residency.
*   **Control plane**: The tenant registry stores `TenantID -> region/tier` and keeps only non-personal metadata centrally. The routing layer sends each tenant to its home region (e.g., tenant-specific subdomains with **Route 53**).
*   **Evidence**: **AWS Config** conformance packs and CloudTrail give auditors proof that data resources exist only in approved regions.
