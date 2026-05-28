# AWS Solution Architect Interview Patterns

A cheat sheet describing 10 core architectural patterns commonly encountered in AWS Solution Architect interviews. Each pattern includes a Mermaid diagram, architectural components, and critical interview talking points.

---

## 1. The CQRS (Command Query Responsibility Segregation) Pattern
Separates read and write operations into different data stores to optimize performance, scalability, and security.

### Mermaid Diagram
```mermaid
graph TD
    Client[Client Browser] -->|Write Requests| WriteAPI[API Gateway - Write API]
    Client -->|Read Requests| ReadAPI[API Gateway - Read API]
    
    WriteAPI -->|Process Commands| WriteDB[(Amazon Aurora RDS)]
    WriteDB -->|CDC Stream| GlueCDC[AWS Database Migration Service DMS]
    GlueCDC -->|Replicate & Format| ReadDB[(Amazon DynamoDB / Elasticsearch)]
    
    ReadAPI -->|Fast Query| ReadDB
```

### Key Interview Points
- **Why AWS?**: Aurora handles transaction compliance (ACID) for writes, while DMS captures modifications and syncs them to DynamoDB (optimized for fast lookups) or OpenSearch (optimized for complex searches).
- **Cons**: Eventual consistency between stores, operational overhead of maintaining replication streams.

---

## 2. Serverless Webhooks & Asynchronous Rate Smoothing
Buffers incoming bursty payloads (like Slack/Stripe webhooks) to prevent downstream transactional databases or external systems from crashing.

### Mermaid Diagram
```mermaid
graph LR
    WebhookSource[External Webhook Source] -->|HTTP POST| APIGW[Amazon API Gateway]
    APIGW -->|Fast JSON Egress| Queue[(Amazon SQS Queue)]
    Queue -->|Controlled Batches| Processor[AWS Lambda Worker]
    Processor -->|Updates| TargetDB[(Amazon RDS Instance)]
```

### Key Interview Points
- **Why AWS?**: SQS holds up to 14 days of backlogged messages. API Gateway directly integrates with SQS (no Lambda needed for ingestion stage) which saves money and limits cold-start timeouts.
- **Scale Buffer**: Lambda can be throttle-limited (Reserved Concurrency) to ensure it only queries the target RDS within its maximum pool limit.

---

## 3. Storage Fan-Out (Pub-Sub to SQS Sinks)
Distributes a single event notification to multiple independent processing queues without the publisher knowing who receives them.

### Mermaid Diagram
```mermaid
graph TD
    UserApp[User Application] -->|Upload Object| Bucket[(Amazon S3)]
    Bucket -->|Event Notification| Topic((Amazon SNS Topic))
    
    Topic -->|Fanout Subscription 1| Queue1[(SQS: Image Resizer)]
    Topic -->|Fanout Subscription 2| Queue2[(SQS: Metadata Extractor)]
    Topic -->|Fanout Subscription 3| Queue3[(SQS: Audit Logging)]
    
    Queue1 --> Lambda1[Lambda Resizer]
    Queue2 --> Lambda2[Lambda Extractor]
    Queue3 --> Lambda3[Lambda Auditor]
```

### Key Interview Points
- **Coupling Mitigation**: Publishers and consumers are fully decoupled. Adding a fourth consumer requires zero changes to the publisher code.
- **Failures**: Message filtering can be set up at the SNS layer to send specific events to specific queues, and individual Dead Letter Queues (DLQs) isolate isolated queue crashes.

---

## 4. Multi-Region Active-Passive (Warm Standby) Disaster Recovery
Maintains a scaled-down duplicate of the system running in a secondary AWS region, ready to scale up rapidly if the primary region goes dark.

### Mermaid Diagram
```mermaid
graph TD
    Client[Client Web App] -->|DNS Resolution| Route53[Amazon Route 53]
    Route53 -->|Active Route| R1_ALB[Primary Region ALB]
    Route53 -->|Standby Route - DNS Failover| R2_ALB[Secondary Region ALB]
    
    subgraph Region_1 [us-east-1 - Primary Active]
        R1_ALB --> R1_Web[EC2 Web Servers]
        R1_Web --> R1_DB[(Aurora Primary)]
    end
    
    subgraph Region_2 [us-west-2 - Secondary Standby]
        R2_ALB --> R2_Web[EC2 Web Standby - Scaled Down]
        R2_Web --> R2_DB[(Aurora Global Replica)]
    end
    
    R1_DB -.->|Aurora Global Database Asynchronous Replication| R2_DB
```

### Key Interview Points
- **RTO/RPO Metrics**: 
  - **RTO (Recovery Time)**: Under 15 mins (mostly DNS propagation + scaling EC2 via Auto Scaling).
  - **RPO (Recovery Point)**: Under 1 second (Aurora Global Database lag).
- **Route 53 Routing**: Failover Routing policy combined with active endpoint health checks.

---

## 5. Saga Pattern (Distributed Transactions orchestration)
Manages distributed consistency across separate microservices using a sequence of local transactions coordinated by a state machine.

### Mermaid Diagram
```mermaid
graph TD
    OrderSvc[Order Service] -->|Start Tx| StepFunc[AWS Step Functions Orchestrator]
    
    StepFunc -->|1. Charge Wallet| PayLam[Lambda: Process Payment]
    PayLam -->|Success| StockLam[Lambda: Reserve Inventory]
    PayLam -->|Fail| CompOrder[Lambda: Cancel Order]
    
    StockLam -->|Success| ShipLam[Lambda: Schedule Delivery]
    StockLam -->|Fail| CompPay[Lambda: Refund Payment]
    
    ShipLam -->|Fail| CompStock[Lambda: Release Stock]
```

### Key Interview Points
- **Why AWS?**: Step Functions keeps track of state, allows retry mechanisms, and automatically initiates compensatory Lambdas (rollback triggers) when a step fails.
- **Alternatives**: Choreo-based Saga (using EventBridge/SNS event chaining). Choreography is simpler to set up initially, but orchestration is far easier to audit and trace.

---

## 6. Microservices Sidecar Pattern
Decouples infrastructural concerns (like service mesh proxying, logging, or metric aggregation) from the core application containers.

### Mermaid Diagram
```mermaid
graph LR
    subgraph ECS_Task [Amazon ECS / EKS Pod]
        AppContainer[App Container: Node.js] <-->|Localhost Traffic| SidecarContainer[Sidecar: AWS App Mesh Envoy Proxy]
    end
    
    SidecarContainer <-->|Remote Mutual TLS| OtherServices[External Microservices]
    SidecarContainer -->|Push Logs| CloudWatch[Amazon CloudWatch]
```

### Key Interview Points
- **Benefits**: Zero impact on core language code to rotate certificates, manage traffic routes, or inject faults.
- **AWS Services**: ECS Task Definitions allow specifying multiple container definitions within a single task, sharing storage volumes and local network interfaces.

---

## 7. Strangler Fig Pattern (Monolith Migration)
Gradually replaces specific pieces of system functionality in a monolithic backend with modern serverless microservices until the monolith is retired.

### Mermaid Diagram
```mermaid
graph TD
    Client[Client App] -->|Queries| CloudFront[Amazon CloudFront]
    CloudFront -->|HTTP /api/v2/orders*| APIGW[Amazon API Gateway]
    CloudFront -->|HTTP /* default| ClassicLB[Classic Load Balancer]
    
    APIGW -->|New Modern Microservice| LambdaOrder[AWS Lambda - Order Processing]
    LambdaOrder --> DynamoDB[(Amazon DynamoDB)]
    
    ClassicLB -->|Old Monolithic Core| LegacyEC2[Legacy EC2 Monolith Cluster]
    LegacyEC2 --> LegacyDB[(Oracle Database)]
```

### Key Interview Points
- **Why AWS?**: CloudFront behaviors route traffic to different target origins based on path mappings. Enables zero-downtime, granular release of individual endpoints.
- **Safety**: Rollback requires changing a single CloudFront route configuration or modifying Route 53 weights.

---

## 8. Outbox Pattern for Reliable Distributed Messaging
Guarantees that a database transaction and its corresponding event publication to a message queue both happen atomically.

### Mermaid Diagram
```mermaid
graph TD
    App[Worker Process] -->|1. Relational Transaction| SQL[(Aurora DB)]
    
    subgraph SQL [Aurora PostgreSQL]
        Table1[Business Table: Orders]
        Table2[Outbox Table: Pending Events]
    end
    
    App -->|Write Order| Table1
    App -->|Write Event| Table2
    
    Debezium[Debezium / DMS CDC Service] -->|2. Stream Changes| Table2
    Debezium -->|3. Publish| Kafka[(Amazon MSK / SNS)]
```

### Key Interview Points
- **Double-Write Problem**: Avoids failures where a record is saved to the database, but network timeout causes the messaging queue to miss the event.
- **AWS Realization**: Database Migration Service (DMS) can read transaction logs directly from RDS databases and forward updates securely to EventBridge or Amazon MSK.

---

## 9. API Gateway Cache vs CloudFront Edge Cache
Optimizes latency by determining where to store content closest to user entry.

### Mermaid Diagram
```mermaid
graph TD
    Client[Client Web Browser] -->|1. Global Cache Request| CloudFront[Amazon CloudFront CDN]
    CloudFront -->|2. API Invalidation Gateway| APIGW[Amazon API Gateway]
    APIGW -->|3. Integration Trigger| Lambda[AWS Lambda Function]
```

### Key Interview Points
- **CloudFront CDN**: Best for static resources (CSS, JS, media files) and publicly accessible read endpoints. Geographically distributed at edge locations worldwide.
- **API Gateway Caching**: Best for caching backend dynamic payloads (like user profiles or catalog searches). Cache is bound to the region where the API Gateway is deployed.
- **Key Limit**: API Gateway cache is billed per hour depending on selected cache size, while CloudFront caching is based on egress traffic volume.

---

## 10. Blue-Green Deployments on EKS (GitOps)
Minimizes software update downtime and limits deployment risks by running two identical production environments concurrently.

### Mermaid Diagram
```mermaid
graph TD
    Developer[Developer] -->|Push Code| GitLab[GitLab / GitHub]
    GitLab -->|Trigger Action| CI[GitHub Actions / CodePipeline]
    CI -->|Build Image & Push| ECR[Amazon ECR Registry]
    CI -->|Update K8s Manifest| GitConfig[Git config repo]
    
    subgraph K8s_Cluster [Amazon EKS Cluster]
        ArgoCD[ArgoCD Controller] -->|Sync Manifest| GitConfig
        
        Router[ALB Ingress Controller] -->|Green Target Group 10%| PodsGreen[New Version Pods]
        Router -->|Blue Target Group 90%| PodsBlue[Stable Version Pods]
    end
```

### Key Interview Points
- **Why ArgoCD / GitOps?**: The infrastructure and deployment versions are declaratively defined in Git. Reversing a bad deployment requires single-click Git reverts.
- **Blue-Green switch**: EKS Ingress routing can dynamically shift percentage-based traffic using AWS Load Balancer Controller attributes.
