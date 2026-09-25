# Awesome AWS Architecture — Reference & Prep Companion

[![AWS Certified Solutions Architect](https://img.shields.io/badge/AWS-Solutions%20Architect-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/certification/certified-solutions-architect-associate/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-all)](http://makeapullrequest.com)

Welcome to the **AWS Solutions Architect Reference & Prep Companion**! 🚀

This repository is a comprehensive, production-grade learning resource designed to help you prepare for **AWS Certified Solutions Architect (Associate & Professional) Exams**, master core cloud architecture principles, and ace senior-level systems design interviews.

Every concept and scenario is documented with **Mermaid architecture diagrams**, deep-dive AWS service mappings, real-world trade-offs, pricing dimensions, and actual interview questions.

---

## 🗺️ Architectural Study Roadmap

To guide your study effectively, follow this structured roadmap from foundational concepts to advanced, enterprise-scale hybrid scenarios:

```
[ Foundation: Well-Architected Framework | Distributed Systems | Networking & APIs ]
                                         │
                                         ▼
                            [ Core Pillars & Concepts ]
                            ├── High Availability & DR
                            ├── Serverless Computing
                            ├── Microservices (EKS/ECS)
                            ├── Event-Driven Systems
                            └── DevOps Fundamentals (Containers, K8s, Observability, Data)
                                         │
                                         ▼
                          [ Advanced Security, Governance & AIOps ]
                            ├── Zero-Trust Networking
                            ├── Multi-Account Strategy
                            └── AIOps Operations (DevOps Guru)
                                         │
                                         ▼
[ Real-World Enterprise & Classic Scenarios (RAG, E-Commerce, WhatsApp, Uber, Multi-Region DR) ]
```

---

## ⚙️ Core AWS Architecture Concepts

Explore detailed conceptual deep-dives covering standard cloud patterns, best practices, common pitfalls, and mock interview questions.

*   **[Well-Architected Framework](concepts/well-architected-framework.md)**: Deep dive into the 6 pillars (Operational Excellence, Security, Reliability, Performance, Cost, Sustainability) mapped to native AWS services.
*   **[Distributed Systems Fundamentals](concepts/distributed-systems-fundamentals.md)**: Master core distributed principles (CAP/PACELC, consistent hashing, consensus/Raft, split-brain, 2PC/Sagas, Bloom filters, and Quad trees) mapped to AWS services.
*   **[Networking & API Fundamentals](concepts/networking-and-api-fundamentals.md)**: Explore Route 53 DNS routing, load balancing algorithms, forward/reverse proxies, transport protocols (TCP/UDP, HTTP/3), API styles (REST/GraphQL/WebSockets), rate limiting, and idempotency.
*   **[High Availability & Disaster Recovery](concepts/high-availability-and-dr.md)**: Master Multi-AZ and Multi-Region strategies, active-passive vs. active-active routing, and calculating RTO/RPO objectives.
*   **[Serverless Architecture](concepts/serverless-architecture.md)**: Build highly scalable architectures using AWS Lambda, API Gateway, DynamoDB, and Step Functions, and learn how to mitigate cold starts.
*   **[Microservices on ECS & EKS](concepts/microservices-on-eks-ecs.md)**: Compare container orchestrators (ECS vs. EKS), task/pod security roles, Envoy sidecar service meshes, and automated deployment strategies (Blue/Green, Canary).
*   **[Event-Driven Architecture](concepts/event-driven-architecture.md)**: Deep dive into asynchronous messaging (SQS, SNS, Kinesis, EventBridge), fan-out patterns, visibility timeouts, and row/message ordering guarantees.
*   **[Security & Compliance](concepts/security-and-compliance.md)**: Master the Shared Responsibility Model, VPC security layers (Security Groups, NACLs), WAF/Shield defenses, and KMS envelope encryption.
*   **[Cost Optimization](concepts/cost-optimization.md)**: Learn how to right-size resources, configure cost-effective EC2 purchasing (Spot vs. Savings Plans), and build automated S3 storage class tiering pipelines.
*   **[Multi-Account Strategy](concepts/multi-account-strategy.md)**: Set up enterprise organizations using AWS Control Tower, centralized log archiving, and Organizations-level Service Control Policy (SCP) guardrails.
*   **[AIOps on AWS](concepts/aiops-on-aws.md)**: Design a predictive operations pipeline using CloudWatch metrics, Amazon DevOps Guru, Lookout for Metrics, and custom SageMaker log clustering and auto-remediation runbooks.
*   **[Hybrid Cloud & Migration](concepts/hybrid-cloud-and-migration.md)**: Master hybrid network connectivity (Direct Connect vs. IPSec VPNs), the 7 Rs of cloud migration, AWS Storage Gateway VM appliances, and AWS MGN.
*   **[Generative AI on AWS](concepts/genai-on-aws.md)**: Build secure serverless LLM applications using Amazon Bedrock, semantic search indexes in OpenSearch Serverless, and Retrieval-Augmented Generation (RAG).
*   **[CI/CD & GitOps Patterns](concepts/cicd-and-gitops.md)**: Deploy applications securely using AWS developer tools (CodePipeline, CodeBuild, ECR), AWS CDK self-mutating pipelines, and pull-based ArgoCD GitOps on EKS.
*   **[Infrastructure as Code with Terraform](concepts/terraform.md)**: Define and manage AWS resources declaratively, mastering remote state backends, DynamoDB state locking, multi-account execution, and zero-downtime resource deployment patterns.

### 🧰 DevOps Fundamentals on AWS

Core DevOps topics (containers, Kubernetes, observability, data layers, Linux, automation) mapped to native AWS services.

*   **[VPC Network Design](concepts/vpc-network-design.md)**: CIDR planning with IPAM, subnet tiers, NAT and VPC endpoints, peering vs. Transit Gateway vs. PrivateLink, centralized inspection, and hybrid DNS with Route 53 Resolver.
*   **[IAM Deep Dive](concepts/iam-deep-dive.md)**: Policy evaluation logic, cross-account access and the confused deputy problem, permission boundaries, ABAC, IAM Identity Center, workload identities, and least-privilege workflows.
*   **[Kubernetes on EKS](concepts/kubernetes-on-eks.md)**: Control plane, workloads, autoscaling (HPA, Karpenter), RBAC with IRSA/Pod Identity, VPC CNI networking, and a kubectl troubleshooting table.
*   **[Containers, Docker & ECR](concepts/containers-docker-ecr.md)**: Namespaces/cgroups, image layers, multi-stage builds, image scanning and signing, and ECS/Fargate/EKS/App Runner selection.
*   **[Observability & Monitoring](concepts/observability-and-monitoring.md)**: Metrics/logs/traces, golden signals, SLI/SLO, CloudWatch, X-Ray, OpenTelemetry, Managed Prometheus and Grafana, and Datadog trade-offs.
*   **[Chaos Engineering & Resilience Testing](concepts/chaos-engineering-and-resilience-testing.md)**: Steady-state hypotheses, game days, and AWS Fault Injection Service experiments with stop conditions.
*   **[Databases on AWS](concepts/databases-on-aws.md)**: SQL/ACID fundamentals, RDS vs. Aurora, DynamoDB design, DocumentDB (MongoDB), ElastiCache, and an engine-selection guide.
*   **[Storage on AWS](concepts/storage-on-aws.md)**: Block vs. file vs. object storage, S3 classes and lifecycle, EBS, EFS, FSx, and Storage Gateway decision guidance.
*   **[Linux & Virtualization on EC2](concepts/linux-and-virtualization-on-ec2.md)**: Linux essentials, performance triage playbooks, hypervisors and Nitro, AMIs, IMDSv2, and Session Manager.
*   **[Configuration Management: Ansible & SSM](concepts/configuration-management-ansible-ssm.md)**: Idempotency, Ansible roles and dynamic inventory, Puppet comparison, Systems Manager, and Ansible vs. Terraform.
*   **[Streaming & Search: Kafka & OpenSearch](concepts/streaming-and-search-kafka-opensearch.md)**: Kafka internals mapped to Amazon MSK, Kafka vs. Kinesis vs. SQS, the Elastic Stack on OpenSearch, and big data processing (EMR, Glue, Athena).
*   **[DevOps Automation & Scripting](concepts/devops-automation-scripting.md)**: Safe Bash, Python/boto3, Go SDK v2, regex, Git workflows, and testing strategy with practical AWS cleanup scripts.
*   **[Multi-Cloud: Azure & GCP to AWS Mapping](concepts/multi-cloud-azure-gcp-to-aws-mapping.md)**: Service mapping tables, IAM and networking model differences, OpenStack overview, and lock-in strategy.

---

## 🏗️ Real-World System Design Scenarios

Each scenario features a comprehensive high-level design walkthrough, a Mermaid diagram, cost estimations, failure modes, and senior-level interview questions.

1.  **[Highly Available E-Commerce Platform](scenarios/01-ha-ecommerce-platform.md)**: Design a multi-AZ, auto-scaling retail platform utilizing ECS Fargate, Aurora PostgreSQL, ElastiCache Redis, S3/CloudFront assets, and SQS-decoupled checkout workers.
2.  **[Zero-Trust Security for Fintech](scenarios/02-zero-trust-fintech.md)**: Protect sensitive transaction APIs using Cognito user authentication, WAF rate-limiting, private VPC endpoints (AWS PrivateLink), and KMS Envelope Encryption.
3.  **[GenAI-Powered Document Q&A System](scenarios/03-genai-document-qa.md)**: Implement a secure, serverless Retrieval-Augmented Generation (RAG) system using Amazon Bedrock, OpenSearch Serverless vector search, and S3 document pipelines.
4.  **[GitOps CI/CD Platform on EKS](scenarios/04-cicd-microservices-eks.md)**: Automate container build pipelines using CodePipeline and ECR, deploying declarative Kubernetes manifests to EKS securely using ArgoCD.
5.  **[SaaS Multi-Tenant Architecture](scenarios/05-saas-multi-tenant.md)**: Bridge isolation models (Silo vs. Pool) by provisioning dedicated tenant accounts via Control Tower and enforcing logical row-level partition security in DynamoDB.
6.  **[Cost-Optimized Analytics Lakehouse](scenarios/06-cost-optimized-data-lake.md)**: Process petabytes of historical analytics cost-effectively using Kinesis Firehose, AWS Glue ETL, Apache Iceberg open table formats on S3, and serverless Amazon Athena SQL queries.
7.  **[Hybrid Cloud Disaster Recovery](scenarios/07-hybrid-cloud-dr.md)**: Achieve RTO < 1 Hour and RPO < 1 Minute warm standby recovery using Direct Connect, Volume Storage Gateway, RDS replicas, and Route 53 DNS failover triggers.
8.  **[Cloud-Native Multi-Region DR Options](scenarios/08-cloud-native-dr-options.md)**: Compare Backup & Restore, Pilot Light, Warm Standby, and Active-Active DR strategies for a Product Catalog app on AWS with cost analyses and detailed failover blueprints.

---

## 📋 Solutions Architect Cheat Sheets

*   **[AWS Services Quick Reference](cheatsheets/aws-services-quick-ref.md)**: Core services quick reference sheet listing primary use cases, anti-patterns (when not to use), and architectural limits.
*   **[Classic System Design on AWS](cheatsheets/classic-system-design-aws.md)**: 10 classic system design interview problems (e.g., URL Shortener, WhatsApp, Spotify, Uber, Web Crawler, Rate Limiter) mapped to production-grade AWS architectures.
*   **[System Design Interview Patterns](cheatsheets/sa-interview-patterns.md)**: 10 core architectural patterns (e.g., CQRS, Saga Orchestration, Strangler Fig Monolith Migration, Outbox database synchronization) with Mermaid diagrams and interview talking points.
*   **[DevOps Architect Interview Prep Guide](cheatsheets/devops-architect-prep.md)**: A structured 5-day study plan covering DevOps CoE, DORA metrics, enterprise Jenkins architectures, AWS multi-account deployment strategies, DevSecOps, SRE, and leadership.
*   **[SRE & DevOps Complete Interview Q&A Guide](cheatsheets/sre-devops-complete-guide.md)**: 128 core SRE, DevOps, AWS, and Kubernetes interview questions and scenarios with detailed, beginner-friendly explanations of all underlying concepts.

### 🎤 Interview Question Banks by Format

Practice each question format interviewers use, not just "design X" and "compare X vs. Y".

*   **[Troubleshooting Scenarios](cheatsheets/troubleshooting-scenarios.md)**: 13 "walk me through it" debugging drills (unreachable EC2, ALB 502/503/504, AssumeRole AccessDenied, Lambda VPC timeouts, S3 403s, stuck Terraform locks) with ordered checklists.
*   **[What Happens When…](cheatsheets/what-happens-when.md)**: Step-by-step deep traces of a URL request through CloudFront/ALB/ECS, `kubectl apply` on EKS, Lambda cold starts, `terraform apply`, S3 SSE-KMS writes, EC2 launches, and `git push` to CodePipeline.
*   **[Incident Response Scenarios](cheatsheets/incident-response-scenarios.md)**: 10 time-boxed runbooks for leaked keys, regional outages, 5× bill spikes, L7 DDoS, ransomware, bad deploys, and compromised instances, plus a blameless postmortem template.
*   **[Spot-the-Bug Config Reviews](cheatsheets/spot-the-bug-config-review.md)**: 12 broken IAM, S3, Dockerfile, Kubernetes, Terraform, CI, Bash, and Lambda configs with the problems and least-privilege fixes.
*   **[Back-of-Envelope Sizing](cheatsheets/back-of-envelope-sizing.md)**: Numbers every architect should know, plus 9 worked estimates (DynamoDB capacity, Kinesis shards, NAT costs, S3 storage growth, Lambda cost, bandwidth).
*   **[Behavioral & Leadership (STAR)](cheatsheets/behavioral-leadership-star.md)**: 12 common behavioral questions with what's really being assessed, model STAR outlines, and pitfalls.
*   **[Rapid-Fire Fundamentals](cheatsheets/rapid-fire-fundamentals.md)**: About 120 one-line recall questions across networking, Linux, AWS services, IAM, containers, CI/CD, observability, and IaC.

Every [system design scenario](#️-real-world-system-design-scenarios) also ends with **Interviewer Follow-Up Drills**: what breaks at 10× traffic, cutting cost by 40%, zero-downtime changes, component failures, and a curveball.

---

## 🎯 Tips for Acing the SA Interview

1.  **Lead with the Well-Architected Framework**: Frame your architectural design decisions around the pillars. Never just draw boxes; explain *why* you prioritized Reliability or Performance over Cost for the target workload.
2.  **State Your Assumptions Clearly**: State the requirements if they are unspecified (e.g., *"I am assuming we target 99.99% availability, which requires active Multi-AZ replication"*).
3.  **Be Explicit about Failure Modes**: No cloud architecture is perfect. Always identify what breaks first (e.g., database connection exhaust, regional outages) and present your mitigations (RDS Proxy, Route 53 DNS failovers) proactively.
4.  **Differentiate Push vs. Pull**: Highlight modern operational patterns like GitOps and serverless event-driven flows rather than basic VM-centric lift-and-shifts.
5.  **Master Your Data Layers**: Be prepared to explain exactly why you chose a relational engine (Aurora) over NoSQL (DynamoDB) based on transactions, data structures, and access patterns.
