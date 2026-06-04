# Serverless Architecture on AWS

Serverless is a cloud execution model where developers write code and the cloud provider fully manages container execution, machine provisioning, OS maintenance, security patching, and elastic scaling.

---

## ⚡ Core Pillars of AWS Serverless

1.  **No Server Management**: No operating system, middleware, or runtime patches to apply.
2.  **Flexible Scaling**: Functions and databases scale automatically in response to ingestion volume.
3.  **Pay-for-Value**: You pay strictly for execution time, memory, or read/write capacities consumed. No idle fees.
4.  **Built-in High Availability**: Serverless systems scale automatically across multiple Availability Zones natively.

---

## 🏗️ Serverless Microservices Reference Architecture

The diagram below illustrates a classic synchronous serverless microservice using API Gateway, Lambda, DynamoDB, and Step Functions.

```mermaid
graph LR
    User[Client Application] -->|HTTPS Requests| APIGW[Amazon API Gateway]
    
    subgraph Synchronous Path
        APIGW -->|Trigger Function| LambdaAuth[Authorizer Lambda]
        LambdaAuth -->|Authorize JWT| Cognito[Amazon Cognito]
        APIGW -->|Process Resource| LambdaCore[Core Core Lambda]
        LambdaCore -->|Queries| Dynamo[(Amazon DynamoDB)]
    end
    
    subgraph Asynchronous Path
        APIGW -->|Trigger Workflow| StepFunc[AWS Step Functions]
        StepFunc -->|Step 1: Check Inventory| LambdaInv[Lambda Inv Check]
        StepFunc -->|Step 2: Charge Customer| LambdaPay[Lambda Payment]
    end
```

---

## Key Serverless Components on AWS

### 1. AWS Lambda
FaaS (Function-as-a-Service) processing events.
*   **Provisioned Concurrency**: Keeps a pre-warmed set of functions ready to respond instantly to eliminate cold starts.
*   **Ephemeral Storage**: Up to 10 GB storage available in `/tmp`.

### 2. Amazon API Gateway
Fully managed service to create, publish, maintain, monitor, and secure APIs.
*   **REST API**: Feature-rich, supports API keys, caching, and rate limiting.
*   **HTTP API**: Up to 70% cheaper and faster than REST APIs, optimized for low-latency Lambda integrations.
*   **WebSocket API**: Allows full-duplex bi-directional communication.

### 3. Amazon DynamoDB
Fully managed NoSQL database with single-digit millisecond latency at any scale.
*   **On-Demand Mode**: Pay-per-request scaling model, excellent for unpredictable workloads.
*   **Provisioned Capacity Mode**: Pre-defined WCU (Write Capacity Units) and RCU (Read Capacity Units) with auto-scaling configured, best for steady-state workloads.

### 4. AWS Step Functions
Low-code visual workflow orchestrator used to build distributed state machines.
*   **Standard Workflows**: Long-running (up to 1 year), visual auditing, exact-once execution.
*   **Express Workflows**: Short-lived (< 5 mins), high-throughput (up to 100k events/sec), at-least-once execution.

---

## 🥶 Cold Starts: Mitigations and Best Practices

A **Cold Start** occurs when Lambda must initialize a new container instance to handle an incoming request. This latency can range from 100ms to several seconds.

### Best Practices to Minimize Cold Starts
*   **Right-size your Memory**: Allocating more memory allocates proportionally more CPU, accelerating boot times.
*   **Optimize Code Packages**: Minify code, exclude unused dependencies, and leverage lightweight frameworks.
*   **Use Provisioned Concurrency**: Pre-allocates execution environments for instant execution.
*   **Natively compile**: Compile code into optimized binaries (e.g., using Rust, Go, or Java with GraalVM).

---

## Common Pitfalls in Serverless Design
*   **Lambda Pinning (Databases)**: Initiating direct connections to a traditional relational database (RDS) inside Lambda. Rapid scaling can exhaust database connection pools. (Mitigation: Use **RDS Proxy**).
*   **Over-permissioning Functions**: Assigning a wildcard `*` IAM policy to Lambda. Each function must utilize a scoped IAM role adhering to least privilege.
*   **Mono-Lambda Anti-Pattern**: Packing an entire backend application (e.g., Express.js monolith) into a single Lambda function. This increases cold starts and degrades performance.

---

## SA Interview Questions on Serverless

### Question 1: How do you prevent a serverless application from exhausting database connections?
**Answer**: 
Unlike containerized microservices that hold connection pools active, Lambda instances spin up and tear down constantly, creating and abandoning database connections rapidly. 
To resolve this:
1.  Use **RDS Proxy**. RDS Proxy pools and shares database connections, reducing database memory usage and connection exhaustion hazards.
2.  Use **DynamoDB** which is designed for serverless architectures and interacts via HTTP requests instead of persistent TCP connections.

### Question 2: When should you choose AWS Step Functions over simple Lambda-to-Lambda chaining?
**Answer**: 
Lambda-to-Lambda chaining is an anti-pattern for complex workflows because:
*   **Error Handling**: Managing failures, timeouts, and retries in custom code is difficult and expensive.
*   **Idle Fees**: Function A must run and wait for Function B to complete, incurring double execution costs.
*   **State Tracking**: State must be passed manually between steps.
Use **AWS Step Functions** to orchestrate workflows, handle errors automatically, manage retries, support human-in-the-loop approvals, and eliminate pay-for-idle during wait states.

### Question 3: What is the difference between API Gateway REST APIs and HTTP APIs?
**Answer**: 
*   **HTTP APIs** are lightweight, designed for low-latency proxy integrations (Lambda, ALB, HTTP backends). They are up to 70% cheaper than REST APIs and lack complex features like API keys, client certificates, or request transformations.
*   **REST APIs** are older and fully featured, offering built-in request/response validation, API keys, cache settings, edge-optimized deployments, and WAF integration. Use HTTP APIs for basic microservice routes and REST APIs for enterprise-controlled entry points.

### Question 4: What are the scaling characteristics of AWS Lambda, and how does instance reuse affect cold starts?
**Answer**: 
AWS Lambda scales differently than traditional virtual machines (EC2) or containers (ECS/EKS):

1.  **Concurrency Model**: An active Lambda instance can only process **one transaction at a time**. If 100 concurrent requests arrive, AWS must spin up 100 separate Lambda instances. This is a key difference from containers, which can handle hundreds of concurrent threads inside a single running container.
2.  **Cold Starts vs. Instance Reuse**: 
    *   *Cold Start*: When a request arrives and no idle instance exists, AWS must allocate compute resources, download the function zip/image, and initialize the runtime. This introduces setup latency (a cold start).
    *   *Instance Reuse*: After a transaction finishes, AWS keeps the Lambda execution container "warm" (active but idle) for a period of time (typically up to a few minutes). Subsequent incoming requests are routed to this warm container immediately, bypassing cold start latency.
3.  **Concurrency Limits**: Scaling is automatic but not infinite. By default, new AWS accounts are limited to a soft limit of 10 concurrent executions (previously 1,000), which can be increased by submitting a support ticket.

### Question 5: How do you choose between Amazon API Gateway and an Application Load Balancer (ALB) as the entry point for your microservices?
**Answer**: 
The choice between API Gateway and an Application Load Balancer depends on the features, scaling patterns, and backend integrations required by your workload:

*   **Choose Amazon API Gateway when**:
    1.  **Serverless/API Management Features are Needed**: You require native authentication (Cognito, IAM, custom authorizers), API key management, rate-limiting/usage plans, request validation, response transformations (VTL), or response caching out of the box.
    2.  **Cross-Account/Region Integrations**: You need to invoke backend resources (like Lambda) hosted in different AWS regions or different AWS accounts.
    3.  **Low/Spiky Traffic (True Serverless Billing)**: You want a pay-per-use model without any minimum hourly fees for idle resources.
    4.  **Canary Deployments**: You want to split traffic for releases natively at the gateway stage.
    *   *Limitations*: Strict timeout limit of **30 seconds** and a default throughput limit of 10,000 requests per second.
*   **Choose an Application Load Balancer (ALB) when**:
    1.  **Kubernetes/EC2 Integrations**: You are routing traffic directly to an EC2 Auto Scaling Group, ECS tasks, or EKS worker nodes (via EKS Ingress Controller) in the same account and region.
    2.  **Long-Running Requests**: Your backend processes take longer than 30 seconds to respond. ALB supports connection timeouts up to **4,000 seconds**.
    3.  **Extremely High/Burst Traffic**: You require scaling beyond 10,000 requests per second without configuration limits (ALB scales virtually infinitely, though pre-warming is recommended for sudden bursts).
    4.  **Layer 7 Path/Host Routing**: You are routing traffic based strictly on DNS hostnames or complex folder paths, and don't need method-based routing (GET vs. POST) at the load balancer level.
    *   *Limitations*: No native rate-limiting (requires WAF), no response caching (requires CloudFront), and incurs base hourly charges even when idle.

### Question 6: How do you choose between Amazon API Gateway and AWS AppSync as the entry point for your serverless applications?
**Answer**: 
The choice between Amazon API Gateway and AWS AppSync primarily comes down to the API paradigm (REST vs. GraphQL), data-fetching flexibility, and real-time synchronization requirements:

*   **Choose AWS AppSync (GraphQL) when**:
    1.  **GraphQL is Required**: Your clients need the flexibility to query specific fields and combine multiple data requests into a single network payload, preventing the over-fetching or under-fetching of data.
    2.  **Real-Time Data Streams**: Your application requires real-time pub-sub capabilities (e.g., chat applications, live dashboards, collaborative workspaces) out of the box. AppSync handles this natively via GraphQL Subscriptions using managed WebSockets.
    3.  **Offline Data Sync**: You are building mobile/web applications (often using AWS Amplify) that require local caching, offline writes, and automatic conflict resolution when the network reconnects.
    4.  **Unified Data Graph**: You want a single schema resolver connecting multiple heterogeneous backends (DynamoDB, Lambda, RDS, HTTP APIs) in one cohesive data layer.
*   **Choose Amazon API Gateway (REST/HTTP) when**:
    1.  **Standard RESTful Microservices**: You are building structured, predictable endpoints (e.g., `/orders`, `/users`) with standard HTTP methods (GET, POST, DELETE).
    2.  **Legacy/Third-Party Integrations**: You are interfacing with traditional REST-based APIs, SaaS tools, or enterprise systems that don't utilize GraphQL.
    3.  **Advanced API Management**: You require robust features like client API keys, custom usage plans, throttling tiers, built-in request/response VTL transformations, and direct integrations with non-database AWS services.
    4.  **Low-Latency Proxying**: You only need to proxy requests straight to a single Lambda function or ALB with minimal overhead, in which case API Gateway HTTP APIs are cheaper and faster.



