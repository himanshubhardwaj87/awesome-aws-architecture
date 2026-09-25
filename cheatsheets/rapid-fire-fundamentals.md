# Rapid-Fire Fundamentals

Short, one-line recall questions for the "lightning round" that many DevOps, SRE, and Solutions Architect interviews open with. Cover the answer column and quiz yourself. Each section links to the deeper concept guide.

> Values marked **(default)** are AWS default quotas or settings that can change or be raised. Verify them against current AWS documentation before quoting them as hard facts.

---

## 🌐 Networking & DNS

Deep dive: [Networking & API Fundamentals](../concepts/networking-and-api-fundamentals.md)

| Question | Answer |
| :--- | :--- |
| What port does DNS use, and when does it use TCP? | Port 53. UDP for normal queries; TCP for zone transfers (AXFR/IXFR) and when a response is too large and comes back truncated (TC bit). |
| What are the TCP handshake steps? | SYN → SYN-ACK → ACK. |
| Is a security group stateful? | Yes. Return traffic is automatically allowed, and security groups support allow rules only. |
| Is a network ACL stateful? | No. It is stateless, evaluates numbered rules in order, supports explicit deny, and needs rules for ephemeral return ports (1024-65535). |
| How many IP addresses does AWS reserve per subnet? | 5 (network address, VPC router, DNS, future use, broadcast). A /24 gives 251 usable. |
| What CIDR sizes can a VPC have? | Between /16 and /28 per CIDR block. |
| Can you put a CNAME at the zone apex? | No (DNS standards forbid it). Use a Route 53 **Alias** record instead. |
| Is VPC peering transitive? | No. A-B and B-C peering does not let A reach C; use Transit Gateway for hub-and-spoke. |
| Internet Gateway vs NAT Gateway? | IGW gives two-way internet access to public subnets; NAT Gateway gives outbound-only access to private subnets. |
| Gateway vs interface VPC endpoint? | Gateway (S3, DynamoDB only) is a route-table entry and free; interface endpoint (PrivateLink) is an ENI with a per-hour and per-GB cost. |
| ALB vs NLB? | ALB is layer 7 (HTTP/HTTPS, path/host routing, WAF); NLB is layer 4 (TCP/UDP/TLS, very high throughput, static IP per AZ). |
| HTTP 502 vs 503 vs 504? | 502 Bad Gateway: invalid response from upstream. 503 Service Unavailable: no healthy capacity. 504 Gateway Timeout: upstream too slow. |

---

## 🐧 Linux

Deep dive: [Linux & Virtualization on EC2](../concepts/linux-and-virtualization-on-ec2.md)

| Question | Answer |
| :--- | :--- |
| SIGTERM vs SIGKILL? | SIGTERM (15) asks a process to exit and can be caught for graceful shutdown; SIGKILL (9) is enforced by the kernel and cannot be caught or ignored. |
| What does load average measure? | Average number of runnable plus uninterruptible (D-state) tasks over 1, 5, and 15 minutes. Compare it with the CPU core count. |
| What is a zombie process? | A process that has exited but whose parent has not called `wait()` to reap it. Fix the parent or kill it so init adopts and reaps the zombie. |
| How do you find what is listening on port 443? | `ss -tulpn \| grep 443` or `lsof -i :443`. |
| `df` shows the disk full but `du` doesn't add up. Why? | A deleted file is still held open by a process. Find it with `lsof +L1` and restart that process. |
| "No space left on device" but `df -h` shows free space? | Inodes are exhausted (many small files). Check with `df -i`. |
| What does `chmod 750` mean? | Owner rwx, group r-x, others no access. |
| Hard link vs symbolic link? | A hard link is another name for the same inode (same filesystem only); a symlink is a pointer to a path and can dangle. |
| How do you check whether the OOM killer fired? | `dmesg -T \| grep -i oom` or `journalctl -k`. |
| How do you see logs for a systemd service? | `journalctl -u <service> -f`. |
| What does high `%wa` (iowait) in `top` mean? | The CPU is idle waiting on disk I/O. Check `iostat -x` and the EBS volume metrics. |
| How do you raise the open file limit? | `ulimit -n` for the session; `/etc/security/limits.conf` or `LimitNOFILE=` in the systemd unit to make it persistent. |

---

## 💻 AWS Compute

Deep dive: [Serverless Architecture](../concepts/serverless-architecture.md) · [Containers, Docker & ECR](../concepts/containers-docker-ecr.md)

| Question | Answer |
| :--- | :--- |
| What is the max Lambda timeout? | 15 minutes (900 seconds). |
| What is Lambda's memory range? | 128 MB to 10,240 MB; CPU is allocated in proportion to memory. |
| What is Lambda's concurrency limit? | 1,000 concurrent executions per region **(default, soft limit)**. |
| How do you remove Lambda cold starts? | Provisioned concurrency; SnapStart for Java, Python, and .NET reduces them. |
| How much warning does a Spot interruption give? | 2 minutes (via instance metadata and EventBridge). |
| What are the three placement group types? | Cluster (low latency, one AZ), Spread (separate hardware, max 7 running instances per AZ per group), Partition (isolated racks for HDFS, Kafka, Cassandra). |
| What happens to instance store data on stop? | It is lost on stop, hibernate, or terminate. It survives a reboot. |
| What does IMDSv2 add? | Session-token-based metadata access (PUT then GET), which blocks SSRF-style credential theft. Enforce it on all instances. |
| What is the default ASG cooldown? | 300 seconds (default). Target tracking uses instance warm-up instead. |
| What is the max Fargate task size? | 16 vCPU and 120 GB memory **(verify current)**. |

---

## 🗄️ Storage

Deep dive: [Storage on AWS](../concepts/storage-on-aws.md)

| Question | Answer |
| :--- | :--- |
| Is S3 strongly consistent? | Yes. Strong read-after-write consistency for PUTs, DELETEs, and LIST operations since December 2020. |
| What is S3's designed durability? | 99.999999999% (11 nines). |
| What is the S3 per-prefix request rate? | 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per prefix. |
| What is the largest single S3 PUT? | 5 GB. Use multipart upload for larger objects (recommended above ~100 MB). |
| What are the S3 minimum storage durations? | Standard-IA and One Zone-IA: 30 days. Glacier Instant and Flexible Retrieval: 90 days. Deep Archive: 180 days. |
| What is the minimum billable object size in IA? | 128 KB. |
| Can you disable S3 versioning? | No, only suspend it once enabled. |
| Object Lock modes? | Governance (privileged users can override) and Compliance (nobody can delete before retention expires, not even root). |
| Is an EBS volume regional or AZ-scoped? | AZ-scoped. Snapshots are regional (stored in S3) and incremental. |
| What is gp3's baseline performance? | 3,000 IOPS and 125 MB/s regardless of size, with IOPS and throughput provisioned independently. |
| Which EBS types support Multi-Attach? | io1 and io2 (Provisioned IOPS), within one AZ, and the filesystem must be cluster-aware. |

---

## 🛢️ Databases

Deep dive: [Databases on AWS](../concepts/databases-on-aws.md)

| Question | Answer |
| :--- | :--- |
| What is DynamoDB's max item size? | 400 KB (attribute names count). |
| What are 1 WCU and 1 RCU? | 1 WCU = one write/s up to 1 KB. 1 RCU = one strongly consistent read/s up to 4 KB (or two eventually consistent reads). |
| GSI vs LSI? | LSI: same partition key, created only at table creation, supports strong consistency, 10 GB limit per partition key. GSI: any key, can be added anytime, eventually consistent only. |
| How long do DynamoDB Streams retain records? | 24 hours. |
| Does DynamoDB TTL delete items immediately? | No. Deletion happens in the background, typically within a few days, so filter expired items in queries. |
| RDS Multi-AZ vs read replica? | Multi-AZ uses synchronous replication for HA (the standby is not readable in the instance deployment); read replicas use asynchronous replication for read scaling. |
| How does Aurora store data? | 6 copies across 3 AZs; writes need a 4/6 quorum and reads a 3/6 quorum. |
| How many Aurora replicas can a cluster have? | Up to 15. |
| What does RDS Proxy solve? | Connection pooling (for example for Lambda storms) and faster, transparent failover. |
| What is the RDS automated backup retention range? | 0 to 35 days (0 disables it); point-in-time restore within that window. |

---

## 🔐 Security & IAM

Deep dive: [Security & Compliance](../concepts/security-and-compliance.md) · [Multi-Account Strategy](../concepts/multi-account-strategy.md)

| Question | Answer |
| :--- | :--- |
| What is the IAM policy evaluation order? | An explicit Deny always wins, then an explicit Allow; otherwise the default is an implicit deny. |
| Do SCPs grant permissions? | No. They only set the maximum permissions for accounts, and they do not apply to the management account. |
| What is a permissions boundary? | The maximum permissions an identity-based policy can grant a user or role; used to delegate IAM creation safely. |
| IAM role vs IAM user? | A role gives temporary credentials via STS and has no long-term keys; a user has long-term credentials. Prefer roles. |
| What is the AssumeRole session duration? | 1 hour by default, configurable up to 12 hours; role chaining is capped at 1 hour. |
| What does cross-account S3 access need? | Permission on both sides: an identity policy in the caller's account and a bucket policy (or ACL) in the owner's account. |
| What is envelope encryption? | KMS encrypts a data key; the data key encrypts the data. Direct KMS Encrypt is limited to 4 KB. |
| How often does KMS rotate customer managed keys? | Automatic rotation is optional; yearly by default, now configurable (90 to 2,560 days). |
| Secrets Manager vs Parameter Store? | Secrets Manager has built-in rotation and a per-secret cost; Parameter Store SecureString is cheaper or free but has no native rotation. |
| GuardDuty vs Inspector vs Macie? | GuardDuty: threat detection from logs. Inspector: vulnerability scanning of EC2, ECR, and Lambda. Macie: PII discovery in S3. |
| How should CI/CD authenticate to AWS? | OIDC federation (for example GitHub Actions to an IAM role) with short-lived credentials, not stored access keys. |

---

## 📦 Containers & Kubernetes

Deep dive: [Kubernetes on EKS](../concepts/kubernetes-on-eks.md) · [Containers, Docker & ECR](../concepts/containers-docker-ecr.md)

| Question | Answer |
| :--- | :--- |
| CMD vs ENTRYPOINT? | ENTRYPOINT is the fixed executable; CMD supplies default arguments that `docker run` args can override. |
| Why use multi-stage builds? | To build in a heavy stage and copy only artifacts into a slim runtime image, which is smaller and has less attack surface. |
| Deployment vs StatefulSet vs DaemonSet? | Deployment: stateless replicas. StatefulSet: stable identity and storage per pod. DaemonSet: one pod per node (agents). |
| Liveness vs readiness vs startup probe? | Liveness failure restarts the container; readiness failure removes the pod from Service endpoints; startup probe delays the other checks for slow starters. |
| What does exit code 137 mean? | Killed by SIGKILL (128 + 9); usually OOMKilled because memory exceeded its limit. |
| CPU limit vs memory limit behavior? | Exceeding a CPU limit causes throttling; exceeding a memory limit gets the container OOM-killed. |
| What does CrashLoopBackOff mean? | The container keeps crashing, and Kubernetes restarts it with exponential back-off. Check `kubectl logs --previous`. |
| HPA vs VPA vs Karpenter? | HPA scales pod count; VPA adjusts pod requests; Karpenter or Cluster Autoscaler adds and removes nodes. |
| How do EKS pods get AWS permissions? | EKS Pod Identity or IRSA (IAM Roles for Service Accounts), never node instance-role sharing. |
| What limits pods per node with the VPC CNI? | ENIs and IPs per instance type, because pods get real VPC IPs. Prefix delegation raises the limit. |
| Where does Kubernetes store cluster state? | etcd (managed by AWS in EKS). |

---

## 🔄 CI/CD & Git

Deep dive: [CI/CD & GitOps](../concepts/cicd-and-gitops.md)

| Question | Answer |
| :--- | :--- |
| `git fetch` vs `git pull`? | Fetch downloads remote refs without changing your branch; pull is fetch plus merge (or rebase) into the current branch. |
| Merge vs rebase? | Merge preserves history with a merge commit; rebase replays commits for linear history. Don't rebase shared branches. |
| `git revert` vs `git reset`? | Revert adds a new commit that undoes a change (safe on shared branches); reset moves the branch pointer, rewriting history. |
| What does `git cherry-pick` do? | Applies a specific commit onto the current branch. |
| What is a detached HEAD? | HEAD points to a commit instead of a branch; new commits are lost unless you create a branch. |
| Blue/green vs canary vs rolling? | Blue/green switches all traffic between two environments; canary shifts a small percentage first; rolling replaces instances in batches. |
| What are the four DORA metrics? | Deployment frequency, lead time for changes, change failure rate, and time to restore service. |
| What is trunk-based development? | Short-lived branches merged to main at least daily, with feature flags hiding incomplete work. |
| What is GitOps? | Git is the source of truth, and an in-cluster agent (Argo CD, Flux) pulls and reconciles desired state. |
| SAST vs DAST vs SCA? | SAST scans source code; DAST attacks the running app; SCA checks third-party dependencies for known CVEs. |

---

## 📈 Observability

Deep dive: [Observability & Monitoring](../concepts/observability-and-monitoring.md) · [AIOps on AWS](../concepts/aiops-on-aws.md)

| Question | Answer |
| :--- | :--- |
| SLI vs SLO vs SLA? | SLI is the measurement; SLO is the internal target; SLA is the contractual promise with penalties. |
| How much downtime does 99.9% allow per month? | About 43 minutes per 30 days (99.99% is about 4.3 minutes). |
| What is an error budget? | 100% minus the SLO. When it is spent, prioritize reliability over features. |
| What are the four golden signals? | Latency, traffic, errors, saturation. |
| RED vs USE? | RED (Rate, Errors, Duration) is for services; USE (Utilization, Saturation, Errors) is for resources. |
| Why prefer p99 over average latency? | Averages hide tail latency that real users feel. |
| Does EC2 publish memory metrics by default? | No. Install the CloudWatch agent for memory and disk usage. |
| EC2 basic vs detailed monitoring? | Basic sends metrics every 5 minutes; detailed sends them every 1 minute. Custom high-resolution metrics can go down to 1 second. |
| What should you alert on? | Symptoms that affect users (SLO burn rate), not every cause such as CPU at 80%. |
| X-Ray vs OpenTelemetry on AWS? | ADOT (AWS Distro for OpenTelemetry) is the vendor-neutral way to instrument; it can export traces to X-Ray. |

---

## 🏗️ Terraform & IaC

Deep dive: [Terraform](../concepts/terraform.md)

| Question | Answer |
| :--- | :--- |
| What is Terraform state for? | It maps configuration to real resource IDs and tracks metadata so Terraform can compute diffs. |
| What is the recommended remote backend on AWS? | S3 with versioning and encryption. Use S3 native locking (`use_lockfile`, Terraform 1.10+); DynamoDB locking is the legacy approach. |
| Is a `sensitive` value safe in state? | No. It is only hidden in CLI output and is still stored in plaintext in state, so protect the backend. |
| `count` vs `for_each`? | `count` indexes by number (removing a middle item shifts others); `for_each` keys by map or set value, which is more stable. |
| What replaced `terraform taint`? | `terraform apply -replace=<address>`. |
| How do you bring existing resources under management? | `import` blocks (Terraform 1.5+) or `terraform import`. |
| How do you rename a resource without destroying it? | A `moved` block (or `terraform state mv`). |
| Useful `lifecycle` arguments? | `prevent_destroy`, `create_before_destroy`, `ignore_changes`. |
| What is drift? | Real infrastructure that differs from code, usually from console changes. `terraform plan` detects it. |
| CloudFormation change set vs StackSet? | A change set previews changes to one stack; a StackSet deploys a stack across many accounts and regions. |

---

## 📨 Messaging & Streaming

Deep dive: [Event-Driven Architecture](../concepts/event-driven-architecture.md) · [Streaming & Search](../concepts/streaming-and-search-kafka-opensearch.md)

| Question | Answer |
| :--- | :--- |
| SQS Standard vs FIFO? | Standard: at-least-once delivery, best-effort ordering, nearly unlimited throughput. FIFO: ordering per message group and exactly-once processing (dedup). |
| What is FIFO throughput? | 300 API calls/s per action, or 3,000 messages/s with batching **(default)**; high-throughput mode is much higher. |
| What is the SQS visibility timeout? | 30 seconds by default, max 12 hours. Set it above your processing time. |
| What is SQS message retention? | 4 days by default, configurable from 60 seconds to 14 days. |
| What is the max SQS long-poll wait? | 20 seconds. |
| What is the SQS max message size? | 1 MiB (raised from 256 KB in Aug 2025). Use S3 pointers for bigger payloads. |
| SNS vs EventBridge? | SNS is high-throughput pub/sub; EventBridge adds content-based routing rules, schema registry, SaaS sources, and archive/replay. |
| SQS vs Kinesis? | SQS: a queue, where messages are deleted after processing. Kinesis: an ordered, replayable stream with multiple consumers per shard. |
| What is the Kinesis shard throughput? | Write: 1 MB/s or 1,000 records/s. Read: 2 MB/s shared, or 2 MB/s per consumer with enhanced fan-out. |
| What is Kinesis retention? | 24 hours by default, up to 365 days. |
| Where does Kafka guarantee ordering? | Within a single partition only. Choose the key so related events share a partition. |
| Why must at-least-once consumers be idempotent? | Duplicates will happen; use idempotency keys or conditional writes so reprocessing has no side effects. |
