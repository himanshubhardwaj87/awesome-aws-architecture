# Scenario 04: CI/CD Platform for Microservices on EKS (GitOps)

## 1. Problem Statement
An enterprise engineering team requires a secure, scalable CI/CD platform to deploy microservices onto an Amazon Elastic Kubernetes Service (EKS) cluster. The deployment process must prevent drift, isolate credentials, and support automated canary rollouts while maintaining separation of concerns between code builds and infrastructure configurations.

---

## 2. Requirements

### Functional
*   Automate code compilation, unit testing, and Docker image generation on Git commits.
*   Store built container images securely with automated vulnerability scanning.
*   Deploy microservices into targeted environments (Dev, Staging, Production) automatically.
*   Allow rollbacks in the event of deployment failures.

### Non-Functional
*   **Security**: No admin Kubernetes credentials stored outside the EKS cluster boundary.
*   **Traceability**: Infrastructure state must match Git configurations exactly (GitOps pattern).
*   **Reliability**: Deploy updates with zero downtime.

---

## 3. Architecture Diagram

![GitOps CI/CD on AWS EKS Architecture](file:///Users/hbhardwaj/Code/awesome-aws-architecture/diagrams/cicd_microservices_eks_architecture.png)

### Interactive Mermaid Blueprint
```mermaid
graph TD
    Developer[Developer] -->|1. Git Push Code| GitApp[GitHub Application Repo]
    
    subgraph CI_Pipeline_Account [Shared Services CI Account]
        GitApp -->|Trigger Build| CodePipeline[AWS CodePipeline]
        CodePipeline -->|Trigger Compilation| CodeBuild[AWS CodeBuild]
        CodeBuild -->|3. Build & Push Image| ECR[(Amazon Elastic Container Registry)]
        CodeBuild -->|4. Push Updated Image Tag| GitOpsConfig[GitHub GitOps Config Repo]
    end
    
    subgraph EKS_Production_Account [EKS Production Account]
        subgraph EKS_Cluster [Amazon EKS Cluster]
            ArgoCD[ArgoCD Controller]
            ALB[AWS ALB Ingress Controller]
            
            ArgoCD -->|6. Auto-sync K8s State| PodsGreen[New Version Pods]
            ArgoCD -->|Deploy Stable Version| PodsBlue[Stable Version Pods]
        end
        
        ArgoCD -->|5. Poll Git Config Changes| GitOpsConfig
        PodsGreen -->|Pull Container Image| ECR
        PodsBlue -->|Pull Container Image| ECR
        
        ALB -->|Canary Route: 10% Traffic| PodsGreen
        ALB -->|Production Route: 90% Traffic| PodsBlue
    end
```

---

## 4. Key AWS Services Used

| Service | Architectural Role | Scoped Purpose |
| :--- | :--- | :--- |
| **Amazon EKS** | Managed Kubernetes. | Hosts microservice applications across private subnets. |
| **AWS CodePipeline** | CI Orchestrator. | Automates build, test, and release pipeline triggers. |
| **AWS CodeBuild** | Serverless Compilation. | Compiles code, runs tests, builds Docker files, and updates Git repositories. |
| **Amazon ECR** | Container Image Registry. | Stores built Docker images securely, running automated vulnerability scans. |
| **AWS ALB Ingress**| Kubernetes Load Balancer.| Dynamically provisions AWS Application Load Balancers for ingress routing. |
| **ArgoCD** | GitOps Reconciliation Engine.| Runs inside EKS to poll Git configs and reconcile EKS clusters to match Git. |

---

## 5. Step-by-Step Design Walkthrough
1.  **Code Commit**: A developer pushes application code changes to the **GitHub Application Repository**.
2.  **Continuous Integration**: **AWS CodePipeline** detects the commit and triggers **AWS CodeBuild**.
3.  **Compilation & Packaging**: CodeBuild compiles the application, runs unit tests, builds a Docker container image, and pushes the image to **Amazon ECR** using a unique git commit hash tag. ECR automatically scans the image for vulnerabilities.
4.  **Configuration Release**: CodeBuild runs a final script that modifies the deployment manifest file (e.g., updating the image tag version in a Helm values file) inside a dedicated **GitHub GitOps Config Repository**.
5.  **GitOps Synchronization**: **ArgoCD**, running inside the **Amazon EKS Cluster**, detects the update inside the GitOps Config repository.
6.  **Declarative Deployment**:
    *   ArgoCD compares the desired Git configuration state with the actual running state of the EKS cluster.
    *   Since a drift exists (new image tag), ArgoCD applies the K8s manifest changes.
    *   EKS pulls the new container image from **Amazon ECR** and launches new pods.
7.  **Traffic Control**: The **AWS ALB Ingress Controller** detects the deployment update and splits traffic (e.g., routing 10% to the newly deployed green version, and 90% to the blue version) to perform canary testing before completing the rollout.

---

## 6. Design Patterns Applied
*   **GitOps Pattern**: Git acts as the single source of truth for infrastructure and deployment states.
*   **Blue/Green Deployments (Canary style)**: Routing a small slice of traffic to a new release to verify stability before deploying fully.
*   **Separation of Concerns**: Decoupling the Application Code Repository from the GitOps Configuration Repository.

---

## 7. Trade-offs

### Pros
*   **Unparalleled Security**: EKS credentials never leave the cluster, minimizing the risk of unauthorized access.
*   **Automatic Drift Reversal**: If a system administrator manually alters an EKS setting in the AWS Console, ArgoCD detects the change and automatically rolls it back to match the Git configuration.
*   **Declarative Rollbacks**: To roll back a bad deployment, simply revert the last commit in the Git configurations repository.

### Cons
*   **High Repository Count**: Requires maintaining separate repositories for code and deployment manifests, increasing configuration overhead.
*   **Reconciliation Lag**: ArgoCD polling intervals introduce a minor delay (usually seconds to minutes) between code merge and active deployment.

---

## 8. When to Use This Pattern
*   Enterprise Kubernetes architectures with strict separation between development and operations teams.
*   High-availability microservice systems requiring zero-downtime canary rollouts.

---

## 9. Cost Estimate

*   **Total Monthly Cost**: ~$500 - $1,200/month.
*   **Key Cost Drivers**:
    *   *Amazon EKS Control Plane Fee*: Fixed cluster fee ($0.10/hour, approx. $73/month).
    *   *CodeBuild Compute Engine*: Charged per build minute. Highly cost-effective for typical build cycles.
    *   *Amazon ECR Storage*: Negligible (charged per GB storage).

---

## 10. Alternatives Considered & Why Rejected
*   **Direct Push Deployments (CodePipeline directly applying kubectl changes)**: Rejected. Requires storing EKS administrative IAM credentials inside the pipeline runner, exposing the cluster to security risks in the event of a pipeline compromise.
*   **Host self-managed Jenkins on EC2**: Rejected due to high administrative overhead. Managing Jenkins servers, updating operating systems, and securing host environments manually violates Operational Excellence principles.

---

## 11. Failure Modes & Mitigations

### 1. Compromised Config Repo
*   **Effect**: Attackers push malicious manifest files to Git, prompting ArgoCD to deploy them.
*   **Mitigation**: Restrict write access to the GitOps Config repository. Enforce signed commits, require multi-party code reviews, and automate pipeline security scans.

### 2. ECR Image Access Failures
*   **Effect**: EKS is unable to download container images, stalling deployments.
*   **Mitigation**: Configure IAM permissions properly using **EKS Node IAM roles** or EKS Pod Identity, allowing tasks to assume the necessary ECR read access.

---

## 12. SA Interview Questions

### Question 1: Explain the difference between Push and Pull deployment pipelines.
**Answer**: 
*   In a **Push Pipeline** (e.g., GitLab CI pushing to EKS), an external CI server connects to the target cluster to run deployment commands. This model is easy to set up initially but requires exposing administrative credentials to the CI server.
*   In a **Pull Pipeline (GitOps)**, an agent (e.g., ArgoCD) runs inside the target cluster and continuously pulls deployment configurations from a Git repository. This model is highly secure, as deployment credentials never leave the cluster boundary, and it supports automatic drift detection and correction.

### Question 2: Why do GitOps architectures use separate repositories for application code and Kubernetes manifests?
**Answer**: 
Using separate repositories provides clear separation of concerns:
1.  **Infinite Loops**: If code and manifests are in the same repository, a pipeline update (like changing an image version tag) triggers a new Git commit, which can run the CI pipeline recursively in an infinite loop.
2.  **Access Control**: Developers can hold write access to the application code repository, while access to the GitOps Configuration repository is restricted to release managers or automated pipelines.
3.  **Clean Audit Trail**: The configuration repository contains a clean, uncluttered audit trail of all infrastructure changes and environment deployment versions.

---

## 🔁 Interviewer Follow-Up Drills

Real interviews push past the first design. Practice defending it against these follow-ups.

### Follow-Up 1: The organization grows 10x in microservices, commits, and deployments. What breaks first, and how do you fix it?
**Answer**: 
*   **First to break**: The **GitOps control loop**. A single **ArgoCD** application controller and repo-server struggle to reconcile thousands of apps, Git polling hits **GitHub API rate limits**, and many **CodeBuild** jobs pushing image tags to one config repo cause push conflicts.
*   **Fixes**:
    1.  **Shard the ArgoCD application controller**, scale out repo-server replicas, and generate apps with **ApplicationSets**.
    2.  Replace polling with **Git webhooks** to ArgoCD. This removes reconciliation lag and API rate-limit pressure.
    3.  Make config-repo updates **retry with rebase**, or open PRs through a bot, or use ArgoCD Image Updater, instead of racing direct pushes.
    4.  Raise **CodeBuild concurrency quotas** (or use reserved capacity fleets), and use **Karpenter** plus pull-through caching so mass pod rollouts don't throttle **ECR** pulls.

### Follow-Up 2: Cut the platform cost by 40%. What do you change, and what do you give up?
**Answer**: 
*   **Worker nodes** (not the $73 control plane) dominate. Use **Karpenter** with **Spot** for stateless services and non-prod, and bin-pack with consolidation. *Give up*: Spot interruptions, which require PodDisruptionBudgets and graceful shutdown.
*   Move to **Graviton**: build multi-arch images on ARM **CodeBuild** runners and run ARM nodes for roughly 20% better price-performance.
*   **Share ALBs** with the Load Balancer Controller's **IngressGroup** instead of one ALB per Ingress.
*   Add **ECR lifecycle policies** to expire untagged and old images, and use **CodeBuild caching** to cut build minutes.
*   Consolidate dev and staging into one cluster with namespaces. *Give up*: Weaker environment isolation. Keep production in its own account.

### Follow-Up 3: How do you upgrade the EKS Kubernetes version with zero downtime?
**Answer**: 
*   **Preferred (GitOps makes this easy): blue/green clusters.**
    1.  Provision a new EKS cluster on the target version and bootstrap **ArgoCD** pointing at the **same GitOps config repo**, so workloads reconcile automatically.
    2.  Shift traffic between the two clusters' ALBs using **Route 53 weighted records** (e.g., 10% then 50% then 100%), then decommission the old cluster.
*   **In-place alternative**:
    1.  Check for removed APIs with **EKS upgrade insights**, then upgrade the control plane.
    2.  Upgrade add-ons (**VPC CNI, CoreDNS, kube-proxy, AWS Load Balancer Controller**).
    3.  Roll the managed node groups (or let Karpenter drift-replace nodes), relying on **PodDisruptionBudgets** and readiness probes so the ALB only routes to ready pods.

### Follow-Up 4: ArgoCD goes down. What's the blast radius, and how do you recover?
**Answer**: 
*   **Blast radius**: **Running workloads are unaffected**. Pods, the ALB, and traffic splits keep serving because Kubernetes is the data plane. What stops is new deployments, **Git-revert rollbacks**, and **drift correction**, so a bad canary can't be rolled back through Git while ArgoCD is down.
*   **Prevention**: Run ArgoCD in **HA mode** (multiple controller, repo-server, and API server replicas plus Redis HA) spread across AZs.
*   **Recovery**: ArgoCD is managed declaratively (the app-of-apps pattern in Git). Reinstalling from the bootstrap manifest restores all Application definitions, and it resyncs from Git with no lost state.
*   **Break-glass**: Keep a tightly scoped, audited IAM role mapped through **EKS access entries** for emergency `kubectl rollout undo`. Commit the same change to Git afterward so ArgoCD doesn't revert it.

### Follow-Up 5: Security mandates that only signed, vulnerability-free images may run in production. How do you enforce it?
**Answer**: 
1.  **Sign in CI**: After CodeBuild pushes to **ECR**, sign the image digest with **AWS Signer** (Notation) or cosign using a KMS-backed key.
2.  **Gate on scanning**: Enable **ECR enhanced scanning (Amazon Inspector)** and fail the pipeline, before the GitOps config commit, on CRITICAL or HIGH findings.
3.  **Enforce at admission**: Run **Kyverno** or **OPA Gatekeeper** in EKS to reject pods whose image isn't signed by the trusted key or isn't referenced by **digest**. This blocks even a compromised config repo from deploying arbitrary images.
4.  **Continuous**: Inspector rescans images already running when new CVEs are published, and findings flow to **Security Hub** so you know which prod workloads to rebuild.
