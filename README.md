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
[ Foundation: Well-Architected Framework ]
                     │
                     ▼
       [ Core Pillars & Concepts ]
       ├── High Availability & DR
       ├── Serverless Computing
       ├── Microservices (EKS/ECS)
       └── Event-Driven Systems
                     │
                     ▼
     [ Advanced Security & Governance ]
       ├── Zero-Trust Networking
       └── Multi-Account Strategy
                     │
                     ▼
  [ Real-World Enterprise Scenarios (RAG, E-Commerce, Data Lakes, Hybrid DR) ]
```

---

## ⚙️ Core AWS Architecture Concepts

Explore detailed conceptual deep-dives covering standard cloud patterns, best practices, common pitfalls, and mock interview questions.

*   **[Well-Architected Framework](concepts/well-architected-framework.md)**: Deep dive into the 6 pillars (Operational Excellence, Security, Reliability, Performance, Cost, Sustainability) mapped to native AWS services.
*   **[High Availability & Disaster Recovery](concepts/high-availability-and-dr.md)**: Master Multi-AZ and Multi-Region strategies, active-passive vs. active-active routing, and calculating RTO/RPO objectives.
*   **[Serverless Architecture](concepts/serverless-architecture.md)**: Build highly scalable architectures using AWS Lambda, API Gateway, DynamoDB, and Step Functions, and learn how to mitigate cold starts.
*   **[Microservices on ECS & EKS](concepts/microservices-on-eks-ecs.md)**: Compare container orchestrators (ECS vs. EKS), task/pod security roles, Envoy sidecar service meshes, and automated deployment strategies (Blue/Green, Canary).
*   **[Event-Driven Architecture](concepts/event-driven-architecture.md)**: Deep dive into asynchronous messaging (SQS, SNS, Kinesis, EventBridge), fan-out patterns, visibility timeouts, and row/message ordering guarantees.
*   **[Security & Compliance](concepts/security-and-compliance.md)**: Master the Shared Responsibility Model, VPC security layers (Security Groups, NACLs), WAF/Shield defenses, and KMS envelope encryption.
*   **[Cost Optimization](concepts/cost-optimization.md)**: Learn how to right-size resources, configure cost-effective EC2 purchasing (Spot vs. Savings Plans), and build automated S3 storage class tiering pipelines.
*   **[Multi-Account Strategy](concepts/multi-account-strategy.md)**: Set up enterprise organizations using AWS Control Tower, centralized log archiving, and Organizations-level Service Control Policy (SCP) guardrails.
*   **[Hybrid Cloud & Migration](concepts/hybrid-cloud-and-migration.md)**: Master hybrid network connectivity (Direct Connect vs. IPSec VPNs), the 7 Rs of cloud migration, AWS Storage Gateway VM appliances, and AWS MGN.
*   **[Generative AI on AWS](concepts/genai-on-aws.md)**: Build secure serverless LLM applications using Amazon Bedrock, semantic search indexes in OpenSearch Serverless, and Retrieval-Augmented Generation (RAG).
*   **[CI/CD & GitOps Patterns](concepts/cicd-and-gitops.md)**: Deploy applications securely using AWS developer tools (CodePipeline, CodeBuild, ECR), AWS CDK self-mutating pipelines, and pull-based ArgoCD GitOps on EKS.

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

---

## 📋 Solutions Architect Cheat Sheets

*   **[AWS Services Quick Reference](cheatsheets/aws-services-quick-ref.md)**: Core services quick reference sheet listing primary use cases, anti-patterns (when not to use), and architectural limits.
*   **[System Design Interview Patterns](cheatsheets/sa-interview-patterns.md)**: 10 core architectural patterns (e.g., CQRS, Saga Orchestration, Strangler Fig Monolith Migration, Outbox database synchronization) with Mermaid diagrams and interview talking points.

---

## 🎯 Tips for Acing the SA Interview

1.  **Lead with the Well-Architected Framework**: Frame your architectural design decisions around the pillars. Never just draw boxes; explain *why* you prioritized Reliability or Performance over Cost for the target workload.
2.  **State Your Assumptions Clearly**: State the requirements if they are unspecified (e.g., *"I am assuming we target 99.99% availability, which requires active Multi-AZ replication"*).
3.  **Be Explicit about Failure Modes**: No cloud architecture is perfect. Always identify what breaks first (e.g., database connection exhaust, regional outages) and present your mitigations (RDS Proxy, Route 53 DNS failovers) proactively.
4.  **Differentiate Push vs. Pull**: Highlight modern operational patterns like GitOps and serverless event-driven flows rather than basic VM-centric lift-and-shifts.
5.  **Master Your Data Layers**: Be prepared to explain exactly why you chose a relational engine (Aurora) over NoSQL (DynamoDB) based on transactions, data structures, and access patterns.
