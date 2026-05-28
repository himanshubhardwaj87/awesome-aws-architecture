# AWS Services Quick Reference

A comprehensive cheat sheet for AWS Solution Architects mapping key services, primary use cases, anti-patterns (when NOT to use), and crucial limits or architectural considerations.

---

## 💻 Compute & Containers

| Service | Primary Use Case | When NOT to Use | Key Limits / Considerations |
| :--- | :--- | :--- | :--- |
| **AWS Lambda** | Event-driven microservices, background processing, short-lived tasks (< 15 mins), glue code. | Long-running processes (> 15 mins), sustained heavy processing, legacy monolithic apps. | - Execution time: max 15 minutes.<br>- Temporary storage (`/tmp`): 512 MB to 10 GB.<br>- Memory: 128 MB to 10,240 MB.<br>- Concurrency limit: default 1,000 per region (soft limit). |
| **Amazon EC2** | Monolithic legacy migrations, custom OS requirements, sustained predictability, high-performance compute. | Short event-driven workloads, highly variable workloads without predictive scaling. | - Scaling speed: minutes (slower than Lambda/Fargate).<br>- Operating burden: requires OS patching, network configuration, and server management. |
| **Amazon ECS** | Microservice container hosting, batch processing, migration of containerized apps, running complex Docker structures. | Zero-overhead lightweight scripting (use Lambda), standard Kubernetes native designs (use EKS). | - Fargate launch type: max 4 vCPUs and 30 GB RAM per task (scalable with EC2 launch type). |
| **Amazon EKS** | Managed Kubernetes deployment, multi-cloud container orchestration, enterprise standard tooling integration. | Low container footprint, simple apps with minimal orchestration needs (use ECS or Fargate). | - High overhead cost compared to ECS ($0.10/hour cluster management fee).<br>- Complex networking (requires CNI plugins like VPC CNI). |

---

## 🗄️ Databases

| Service | Primary Use Case | When NOT to Use | Key Limits / Considerations |
| :--- | :--- | :--- | :--- |
| **Amazon RDS** | Traditional relational apps (PostgreSQL, MySQL, SQL Server), ACID transactional consistency, complex SQL queries. | Low-latency key-value requirements, massive unstructured storage, petabyte-scale write-heavy analytics. | - Max storage limit: 64 TB (PostgreSQL/MySQL).<br>- Requires database connection pooling (e.g., RDS Proxy) for serverless integrations. |
| **Amazon Aurora** | Enterprise-grade HA relational databases, auto-scaling up to 128 TB, global scale active-passive configurations. | Budget-conscious minimal workloads, non-relational simple key-value lookups. | - Connection limits depend on instance sizes.<br>- Global database replication is asynchronous (sub-second lag). |
| **Amazon DynamoDB** | Ultra-low latency (<10ms) document and key-value store, infinite scale, serverless transactional apps. | Highly relational data with complex joins, ad-hoc aggregation queries, heavy analytical reporting. | - Max item size: 400 KB.<br>- Partition limits: 1000 WCU or 3000 RCU per partition before auto-partitioning occurs. |
| **Amazon ElastiCache** | Microsecond latency in-memory data cache (Redis/Memcached), session store, pub-sub messaging, leaderboards. | Permanent persistent data store where no data loss is tolerable. | - Cluster configurations must plan for cluster resizes without downtime using Redis cluster mode. |
| **Amazon Redshift** | Petabyte-scale enterprise data warehousing, complex OLAP analytics queries, BI reporting dashboards. | High-frequency low-latency transaction processing (OLTP), real-time unstructured data ingestion. | - Concurrency Scaling handles bursty spikes, but has a pricing threshold after free credits. |

---

## 📁 Storage

| Service | Primary Use Case | When NOT to Use | Key Limits / Considerations |
| :--- | :--- | :--- | :--- |
| **Amazon S3** | Durable, infinitely scalable object storage, static website hosting, raw data lake staging, backup/archival. | High-performance filesystem requirements, block storage for OS, frequently updated file portions. | - Max object size: 5 TB.<br>- Max single upload: 5 GB.<br>- Rate limits: 3,500 PUT/POST/DELETE and 5,500 GET requests per second per prefix. |
| **Amazon EBS** | High-performance persistent block storage for EC2 instances, boot volumes, transactional DB volumes. | Shared storage across multiple independent EC2 instances (use EFS instead, except EBS Multi-Attach on io1/io2). | - Bound to a single Availability Zone (requires snapshots to move data across AZs). |
| **Amazon EFS** | Scalable, shared elastic file storage for EC2 and ECS (Linux) supporting POSIX concurrent access. | Windows filesystem workloads (use FSx for Windows), ultra-low latency sub-millisecond DB files (use EBS). | - Throughput modes: Elastic, Provisioned, or Bursting.<br>- Mountable from on-premises via VPN or Direct Connect. |

---

## 🌐 Networking & Content Delivery

| Service | Primary Use Case | When NOT to Use | Key Limits / Considerations |
| :--- | :--- | :--- | :--- |
| **Amazon Route 53** | Highly available, scalable DNS, domain registration, traffic routing (latency, geolocation, failover). | Simple internal hostname resolution within a single private subnet (use private Route 53 zones instead). | - Max health checks per account: 200 (default, adjustable). |
| **Amazon CloudFront** | Low-latency static/dynamic asset distribution (CDN), edge SSL termination, API acceleration, web protection. | Strictly internal private enterprise apps where data must not egress to public edge locations. | - Max file size: 20 GB.<br>- Cache invalidation: takes minutes; use versioned query strings or file paths instead. |
| **AWS PrivateLink** | Safe private API exposing without internet routing, connecting customer VPCs to SaaS endpoints. | Bulk large-scale multi-directional subnet bridging (use VPC Peering or Transit Gateway instead). | - Sub-second latency overhead.<br>- Billed per VPC endpoint per hour + data processed. |

---

## 🔄 Messaging & Events

| Service | Primary Use Case | When NOT to Use | Key Limits / Considerations |
| :--- | :--- | :--- | :--- |
| **Amazon SQS** | Decoupling microservices, asynchronous transaction staging, rate smoothing (buffering) to protect DBs. | Real-time event broadcasting to multiple subscribers, strict long-term retention of messages. | - Max message size: 256 KB.<br>- Retention period: 1 minute to 14 days (default 4 days).<br>- SQS Standard: virtually unlimited throughput. SQS FIFO: 3,000 requests/sec with batching. |
| **Amazon SNS** | Pub-sub message publishing to millions of endpoints (HTTP, Email, SMS, SQS), fan-out notification pattern. | Point-to-point guaranteed storage with processing queue logic (use SNS + SQS in tandem). | - Max message size: 256 KB.<br>- Delivery retry policies can be customized per endpoint type. |
| **Amazon Kinesis** | Real-time streaming ingestion of large datasets (clickstreams, log aggregation, IoT data), analytics processing. | Simple asynchronous task queueing, long retention durable storage. | - Kinesis Data Streams: Data record retention from 24 hours up to 365 days.<br>- Scaling done by adjusting shard counts (On-Demand or Provisioned modes). |
| **Amazon EventBridge** | Serverless event bus connecting application events, SaaS integrations, scheduled cron jobs, schema registry. | High-frequency telemetry ingestion exceeding Kinesis streams scale (>100k events/sec). | - Default throughput limit: 10,000 events/second (adjustable). |
