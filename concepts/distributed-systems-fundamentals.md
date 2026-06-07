# Distributed Systems & General System Design Fundamentals on AWS

## 🌐 Introduction
In distributed computing, systems run across multiple machines (nodes) communicating over a network. While distributed architectures enable horizontal scale and high availability, they introduce latency, partial failure, and concurrency challenges. 

This guide covers core distributed systems concepts and maps them directly to AWS services and architectural patterns.

---

## 🏛️ System Design Core Metrics

Reliability and availability must be designed intentionally, understanding where failures can occur.

| Metric | Definition | Measurement / Formulation | Mitigation on AWS |
| :--- | :--- | :--- | :--- |
| **Scalability** | The system's ability to handle growing workloads by adding resources. | $O(N)$ resource scaling under load | Auto Scaling, Elastic Load Balancing, DynamoDB Auto-Partitioning. |
| **Availability** | The percentage of time a system remains operational and responsive. | $\frac{\text{Uptime}}{\text{Uptime} + \text{Downtime}}$ | Multi-AZ deployments, Route 53 DNS Failover, self-healing architectures. |
| **Reliability** | The probability that a system performs its required function under stated conditions for a specified period. | Mean Time Between Failures (MTBF) | Fault isolation (Cell-based architecture), active health checks, error-retries. |

### 1. Single Point of Failure (SPOF)
A Single Point of Failure (SPOF) is any component, dependency, process, or decision point whose failure halts the entire system or disables a critical user flow. 

#### What Makes Something a SPOF?
A component becomes a SPOF when three criteria are met:
1. **Critical Path Dependency**: A key system flow depends directly on its operations.
2. **No Working Alternative**: There is no fallback mechanism or automated failover when it fails.
3. **Unacceptable Business Impact**: Its failure causes system outages, severe data loss, or halts core user journeys (e.g., checkout).

> [!NOTE]
> **Flows over Components**: SPOF analysis should focus on **user flows** rather than just counting servers. For example, if a recommendation engine fails but clients can still complete checkouts, that engine is not a SPOF for the critical checkout flow.
> However, if a cache cluster fails and subsequent database queries overwhelm the database, the cache is a SPOF in a cascading failure path.

#### Common SPOFs in Cloud Architectures & AWS Mitigations

| Area | Common Cloud SPOF | Failure Impact | AWS Native Mitigation |
| :--- | :--- | :--- | :--- |
| **Traffic Entry** | Single DNS server or single Load Balancer. | Clients cannot reach endpoints. | **Route 53 Multivalue/Failover** routing; **Application Load Balancers (ALBs)** pre-provisioned across multiple AZs. |
| **Compute** | Single EC2 instance or ECS task. | Traffic fails when that node fails. | **Auto Scaling Groups (ASGs)** with a minimum capacity $\ge 2$ spanning multiple AZs. |
| **Data** | Single-node relational database. | Complete halt of read/write queries. | **Amazon RDS Multi-AZ** (synchronous replica auto-failover) or **Aurora Global Database**. |
| **Cache** | Single cache cluster with no fallback. | Database overload after cache crash. | **ElastiCache Redis** cluster with Multi-AZ auto-failover and replication groups. |
| **Messaging** | Single broker queue or event hub. | Message production blocks, processing stops. | **Amazon SQS** (serverless, highly redundant by default) or **Amazon MSK Multi-AZ** deployments. |
| **Configuration** | Central configuration file or service. | Bootstrapping and system settings updates block. | **AWS AppConfig** with rollout controls and validation checks. |
| **Secrets** | Hardcoded secrets or single secrets path. | Bootstrapping or authentication fails. | **AWS Secrets Manager** with multi-region replication and rotation keys. |
| **Network** | Single NAT Gateway or VPN tunnel. | Private subnets lose internet/hybrid access. | Deploying **one NAT Gateway per AZ** and configuring redundant VPN tunnels. |

#### How to Identify SPOFs
1. **Dependency Mapping**: List all runtime dependencies (databases, third-party APIs, CDNs, and network routes) involved in critical flows (e.g., login, checkout).
2. **Distinguish Runtime vs. Recovery Graphs**:
   * *Runtime Graph*: What is needed to serve active traffic.
   * *Recovery Graph*: What is needed to repair the system (deploy pipelines, certificate rotation, secrets). If deployment/recovery relies on a single service or tool, that tool is a recovery SPOF.
3. **Chaos Engineering**: Regularly inject faults using tools like **AWS Fault Injection Simulator (FIS)** (e.g., dropping database connections, increasing network latency) to verify system resilience.

#### Redundancy Trade-Offs
Eliminating SPOFs introduces trade-offs:
* **Cost**: Running active-active instances across regions or maintaining Multi-AZ database mirrors increases bills.
* **Complexity**: Designing data replication streams introduces state synchronization challenges and potential consistency issues.
* **Safety Rules**: Some critical components should fail closed rather than open. If a security policy checker fails, rejecting requests (denying access) is safer than failing open, even if it impacts availability.


---

## 🔄 Consistency Models

When replicating data across multiple nodes, we must define how updates propagate to clients:

```
[Strong Consistency] ◄────────────────────────────────────────► [Eventual Consistency]
  - Immediate visibility                                          - Delayed propagation
  - High read latency (locking)                                  - Low read latency (local read)
  - RDS Multi-AZ Sync                                            - DynamoDB replicas (default)
```

1.  **Strong Consistency (Linearizability)**: Once a write completes, all subsequent reads return the value of that write.
    *   *AWS Realization*: **Amazon RDS/Aurora Multi-AZ** (synchronous commits to standby database) or **DynamoDB Consistent Reads** (which query the leader node of the 3-way storage replica).
2.  **Sequential Consistency**: Operations occur in a global order, and all replica nodes agree on this sequence, though they might be slightly behind real time.
3.  **Eventual Consistency**: Replicas converge on the same value eventually if no new writes occur.
    *   *AWS Realization*: **DynamoDB Global Tables** (asynchronous cross-region replication) or **S3 Cross-Region Replication** (CRR).
4.  **Monotonic Reads**: If a client reads value $v_1$ at time $t_1$, they will never read an older value $v_0$ at a later time $t_2$.
5.  **Read-Your-Writes**: A client will always see their own updates immediately, even if other users see them eventually.

---

## 📊 CAP vs. PACELC Theorems

### 1. CAP Theorem
In a distributed data store, you can guarantee at most two of the following three properties during a network partition ($P$):
*   **Consistency (C)**: Every read receives the most recent write or an error.
*   **Availability (A)**: Every non-failing node returns a non-error response (without guarantee that it contains the most recent write).
*   **Partition Tolerance (P)**: The system continues to operate despite arbitrary message loss or system partitions.

```
                      Consistency (C)
                            /\
                           /  \
                          /    \
            RDS Multi-AZ /  P   \ DynamoDB Strong Reads
             (CP/Sync)  /________\  (AP/Async Default)
         Availability (A)        Partition Tolerance (P)
```

> [!IMPORTANT]
> Since network partitions ($P$) are inevitable in physical infrastructure, distributed databases must trade off between Availability ($A$) or Consistency ($C$). 
> *   **CP Systems**: Choose consistency over availability (e.g., RDS Multi-AZ halts writes if synchronous commit fails).
> *   **AP Systems**: Choose availability over consistency (e.g., DynamoDB default reads return immediately from local replicas even if cross-region sync is delayed).

### 2. PACELC Theorem
An extension of CAP: **P**artition ($P$) -> choose **A**vailability ($A$) or **C**onsistency ($C$); **E**lse ($E$) -> trade off **L**atency ($L$) or **C**onsistency ($C$).
*   **DynamoDB Mapping**: Normally (Else), DynamoDB is a **PA/EL** system by default. It prioritizes low **L**atency ($L$) by serving local replica reads rather than enforcing immediate **C**onsistency ($C$), but can be configured as **PC/EC** if Strong Consistent Reads are requested.

---

## 📍 Consistent Hashing

Traditional hashing ($Node = Hash(Key) \pmod N$) requires rehashing all keys when nodes ($N$) are added or removed, causing massive database disruptions. **Consistent Hashing** maps both keys and nodes onto a circular ring, minimizing key relocation during scaling.

```mermaid
graph TD
    subgraph Hashing Ring (0 to 2^32 - 1)
        Ring((Circle Ring))
        NodeA[Node A at 100k] -->|Saves Keys 0-100k| Ring
        NodeB[Node B at 200k] -->|Saves Keys 100k-200k| Ring
        NodeC[Node C at 300k] -->|Saves Keys 200k-300k| Ring
    end
    
    NewNodeD[New Node D added at 150k] -.->|Only rehashes keys 100k-150k from Node B| NodeB
```

*   **Virtual Nodes (Vnodes)**: Single physical nodes are mapped to multiple positions on the ring to distribute the load evenly.
*   **AWS Mapping**: 
    *   **Amazon DynamoDB**: Uses consistent hashing under the hood to partition data across physical SSD storage drives based on the hash of the Partition Key.
    *   **Amazon ElastiCache Redis (Clustered Mode)**: Implements 16,384 logical hash slots distributed across master nodes, which is functionally equivalent to consistent hashing for slot redistribution.

---

## 🗳️ Consensus Protocols & Leader Election

Distributed consensus is the process of getting multiple nodes to agree on a single data value or system state (e.g., who is the master/leader).

### 1. Paxos and Raft
*   **Paxos**: The classic consensus protocol based on Proposers, Acceptors, and Learners. Highly complex to implement.
*   **Raft**: A modern consensus protocol designed for readability. Deconstructs consensus into explicit phases: Leader Election, Log Replication, and Safety.
*   **AWS Realization**:
    *   **Amazon MSK (Kafka)**: Uses Raft (KRaft mode) or ZooKeeper (which uses Zab, a Paxos-variant protocol) for metadata consensus and partition leader election.
    *   **Amazon Route 53 DNS Resolver**: Internal consensus ensures updates replicate reliably across international DNS root name servers.

### 2. Leader Election and Split-Brain
When a cluster loses its leader, a new leader must be elected. If network partition separates the cluster, both halves might elect different leaders, leading to data corruption (**Split-Brain**).
*   **AWS Mitigation**: Use quorum majorities ($\lfloor N/2 \rfloor + 1$) for consensus, or offload leader election using **Amazon DynamoDB Lock Client** (lease-based distributed locking).

---

## 🤝 Distributed Transactions & Replication

### 1. Two-Phase Commit (2PC) vs. Three-Phase Commit (3PC)
*   **2PC**: A blocking protocol where a coordinator polls participants to Prepare, and then Commit. If the coordinator crashes mid-process, resources remain locked.
*   **3PC**: A non-blocking variant that introduces a "Pre-Commit" state to allow recovery if the coordinator fails, though it is still vulnerable to network partitions.
*   **Modern Alternative**: Because 2PC/3PC limit scale due to locking, modern systems use the **Saga Pattern** (orchestrated by **AWS Step Functions**) using asynchronous compensating transactions for rollbacks.

### 2. State Sync: Gossip Protocol & Vector Clocks
*   **Gossip Protocol**: A peer-to-peer communication method where nodes periodically share information with random peers to disseminate cluster status.
    *   *AWS Mapping*: **ECS Service Discovery** nodes check in via DNS gossip, and DynamoDB internal partitions monitor cluster health using similar heartbeat gossip.
*   **Vector Clocks**: Algorithms used to generate partial ordering of events and detect concurrent write conflicts in leaderless database replication.
    *   *AWS Mapping*: DynamoDB uses version vectors internally to resolve conflict updates in multi-region replication.
*   **Conflict-Free Replicated Data Types (CRDTs)**: Special data structures (like registers, sets, or counters) that can be updated independently and merged without conflicts.
    *   *AWS Mapping*: Used in AWS AppSync Delta Sync configurations to merge concurrent client updates on offline mobile clients.

---

## 🔍 Advanced Data Structures in System Design

### 1. Bloom Filters
A space-efficient probabilistic data structure used to test whether an element is a member of a set.
*   **Key Characteristics**: **No false negatives**, but **possible false positives** (can tell if something is *definitely not* in a set, or *probably* in it).
*   **AWS Implementation**: Used inside **DynamoDB** and **ElastiCache Redis** to prevent expensive disk reads/database hits for non-existent record lookups.

### 2. Quad Trees
A tree data structure in which each internal node has exactly four children. Used to partition a two-dimensional space (e.g., latitude and longitude).

```
                      [ Root - Global Map ]
                       /    |     \     \
                      /     |      \     \
                   [NE]    [NW]   [SE]   [SW]
```

*   **Use Cases**: Location-based services (e.g., Yelp, Uber) to query points of interest within a bounded spatial area.
*   **AWS Mapping**: Implementing spatial indexes on **Amazon Aurora PostgreSQL** (using PostGIS extension) or using **Amazon Location Service** for location queries.

---

## 📋 Distributed Systems Interview Q&A

### Question 1: How does Amazon DynamoDB partition data as your database scales?
**Answer**:
DynamoDB uses **Consistent Hashing**. Under the hood, DynamoDB maps partition keys to an internal hashing space of size $2^{128}$. The hashing space is split into ranges, with each range assigned to a specific physical storage node. 
When writes exceed 1,000 WCUs or storage exceeds 10 GB, DynamoDB automatically splits partitions. Because it uses consistent hashing, splitting a partition only requires shifting a subset of keys to the new partition node, rather than reorganizing the entire dataset.

### Question 2: In the context of the CAP Theorem, how does Amazon Aurora Global Database handle a regional failure?
**Answer**:
Aurora Global Database operates as a single-master system across regions. The primary region serves as the writer, replicating asynchronously to secondary read-only regions (with replication lag typically < 1 second). 
*   **Normal Operations**: It functions as a **CP system** for writes (enforcing ACID consistency within the primary region) while offering low-latency AP reads in secondary regions.
*   **Regional Failover**: In the event of a primary region outage, the system must promote a secondary region to write master. This is a deliberate CP decision: writes are temporarily blocked (violating availability) until the replica is promoted, ensuring zero split-brain data collisions.

### Question 3: Explain how a Bloom Filter saves costs in a distributed caching layer on AWS.
**Answer**:
Consider a Web Crawler or URL Shortener designed to lookup millions of records. If a requested key does not exist, looking it up in the database requires an expensive I/O operation (or cache miss penalty).
By placing a **Bloom Filter in ElastiCache Redis**, the system checks if the key exists *before* hitting the database. If the Bloom filter returns `false`, the app immediately returns a 404 response without executing any database query, preserving write/read IOPS and reducing operational costs.
