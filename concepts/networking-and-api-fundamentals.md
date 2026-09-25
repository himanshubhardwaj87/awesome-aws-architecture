# Networking & API Fundamentals on AWS

## 🌐 Introduction
In distributed architectures, networks serve as the nervous system connecting services. Designing highly available and responsive systems requires a deep understanding of network layers, proxy servers, load balancing algorithms, communication protocols, and API integration designs.

This guide outlines core networking and API fundamentals, mapping them directly to native AWS services.

---

## 🗺️ The Domain Name System (DNS) & Route 53

DNS is a hierarchical, decentralized naming system that translates human-readable hostnames (e.g., `example.com`) into machine-readable IP addresses.

```
                  [ Client Browser ]
                          │
            1. Resolve?   ▼   4. IP Address Resolved
                  [ Route 53 DNS ]
                   /      │       \
       2. Query   /       │        \  3. Query
                 ▼        ▼         ▼
             [Root]    [TLD]     [Authoritative]
             Servers   Servers     Name Servers
```

### Route 53 Routing Policies
AWS Route 53 is a highly available and scalable Domain Name System (DNS) web service. It supports several routing policies to match application needs:

*   **Simple Routing**: Resolves a single resource record (e.g., one IP address) for a domain.
*   **Weighted Routing**: Directs traffic to multiple resources in proportions you specify (e.g., 90% to Blue, 10% to Green for deployments).
*   **Latency-based Routing**: Directs requests to the AWS region that provides the lowest network latency for the end user.
*   **Failover Routing**: Configures active-passive failover, routing traffic to a secondary disaster recovery endpoint if the primary endpoint fails its health checks.
*   **Geolocation Routing**: Routes traffic based on the physical location of the user (e.g., EU users are routed to resource instances in Frankfurt).
*   **Geoproximity Routing**: Routes traffic based on the geographic location of resources and optionally shifts traffic boundaries using bias factors.
*   **Multivalue Answer Routing**: Returns up to eight healthy resource records in response to DNS queries, allowing clients to distribute traffic across endpoints.

---

## ⚖️ Load Balancing & Proxies

### 1. Forward Proxy vs. Reverse Proxy
*   **Forward Proxy**: Acts on behalf of a group of client machines. It intercept requests from clients, masks client identities, and forwards requests to the internet.
*   **Reverse Proxy**: Acts on behalf of backend application servers. It accepts incoming public client traffic, routes requests to appropriate backend nodes, and manages TLS handshake, caching, and rate limiting.
    *   *AWS Mapping*: **Amazon CloudFront** and **Application Load Balancers (ALBs)** serve as managed reverse proxies.

### 2. Load Balancing Algorithms
Load balancers distribute traffic across a pool of backend servers using different algorithms:
*   **Round Robin**: Sequentially routes requests to each server. Simple but ignores server load.
*   **Least Connections**: Routes requests to the server handling the fewest active sessions. Ideal for long-running connections.
*   **IP Hashing**: Hashes the client's IP address to map them consistently to the same backend server (session affinity).
*   **Path-based/Rule-based**: Evaluates request properties (HTTP path, headers, query parameters) to select targets.

### 3. Elastic Load Balancing (ELB) Matrix
AWS provides specialized load balancers across the network stack:

| ELB Type | OSI Layer | Focus / Characteristics | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Application (ALB)** | Layer 7 (HTTP/S) | Path/host routing, redirects, HTTP header inspections, TLS termination. | Microservices (ECS/EKS), Web Apps. |
| **Network (NLB)** | Layer 4 (TCP/UDP) | Millions of requests/sec, ultra-low latency, preserves source IPs, static IPs. | High-performance API egress, gaming. |
| **Gateway (GLB)** | Layer 3 (IP packets)| Routes traffic through third-party virtual appliances (firewalls, IDSs). | Centralized VPC security inspection. |

---

## 📨 Protocols & Transport Layers

### 1. TCP vs. UDP
*   **Transmission Control Protocol (TCP)**: Connection-oriented. Guarantees packet delivery, packet ordering, and implements congestion control (via a three-way handshake).
    *   *AWS Target*: Standard ALB workloads, API Gateways, database queries (RDS).
*   **User Datagram Protocol (UDP)**: Connectionless. Lightweight, does not guarantee delivery or packet ordering. Optimized for speed and low overhead.
    *   *AWS Target*: Gaming backends, live media streaming, DNS resolvers, routed through **NLB** or **AWS Global Accelerator** (utilizing UDP listener rules).

### 2. HTTP Progressions
*   **HTTP/1.1**: Introduces Keep-Alive to reuse TCP connections, but suffers from **head-of-line blocking** (subsequent requests must wait for preceding responses to complete).
*   **HTTP/2**: Introduces binary framing and **multiplexing** over a single TCP connection, allowing concurrent requests and server push.
*   **HTTP/3**: Replaces TCP with **QUIC** (which runs over UDP), eliminating head-of-line blocking at the transport level during packet loss.
    *   *AWS Support*: CloudFront and ALBs support HTTP/2 and HTTP/3 natively.

---

## 🛠️ API Architectural Styles

Different communication patterns dictate API designs:

```
[REST API]             [GraphQL]              [WebSockets]             [Webhooks]
├── Pull Pattern       ├── Pull Pattern       ├── Full Duplex          ├── Push Pattern
├── Stateless          ├── Single Endpoint    ├── Persistent TCP       ├── Asynchronous Event
└── HTTP Verbs         └── Custom Schema      └── Real-time Chat       └── Stripe / Github
```

1.  **REST (Representational State Transfer)**:
    *   *Core*: Resource-based, stateless, leverages standard HTTP methods (`GET`, `POST`, `PUT`, `DELETE`).
    *   *AWS Realization*: **Amazon API Gateway** (REST/HTTP APIs) or Application Load Balancers routing to ECS/Lambda.
2.  **GraphQL**:
    *   *Core*: Single endpoint (`/graphql`). Allows clients to define the exact shape of data they need, eliminating over-fetching and under-fetching.
    *   *AWS Realization*: **AWS AppSync** (fully managed GraphQL engine with real-time subscriptions).
3.  **WebSockets, Webhooks, and WebRTC**:
    *   **WebSockets**: A persistent, bidirectional, full-duplex TCP connection established via an HTTP handshake.
        *   *AWS Realization*: **API Gateway WebSocket APIs** manage connection states dynamically, executing Lambdas during events.
    *   **Webhooks**: Event-driven push notifications. A service makes an asynchronous HTTP POST request to a client-defined URL when an event occurs.
        *   *AWS Realization*: **Amazon EventBridge API Destinations** to push events to third-party APIs.
    *   **WebRTC**: A peer-to-peer framework enabling real-time voice, video, and data transmission directly between browsers.
        *   *AWS Realization*: **Amazon Kinesis Video Streams with WebRTC** provides secure signaling servers to establish peer connections.

---

## 🚦 Rate Limiting & Traffic Management

Rate limiting prevents API exhaustion, mitigates DDoS attacks, and controls API consumption rates.

### 1. Rate Limiting Algorithms
*   **Token Bucket**: A bucket is filled with tokens at a constant rate. Requests consume a token. If the bucket is empty, the request is rate-limited. Supports bursty traffic.
*   **Leaky Bucket**: Requests enter a queue. The queue drains to the server at a constant, steady rate. Smooths out traffic spikes.
*   **Sliding Window Log**: Tracks timestamps of all incoming requests in a log. Evaluates log entries dynamically to compute exact rate limits. Memory-intensive.
*   **Sliding Window Counter**: Combines request counts from the previous window and the current window to estimate current rates with low memory overhead.

### 2. Rate Limiting on AWS
*   **API Gateway Throttling**: Restricts client requests using Token Bucket algorithms. Configured at the stage or route level (e.g., 10,000 requests/sec limit, 5,000 burst).
*   **AWS WAF Rate Limiting**: Evaluates IP request rates over a sliding 5-minute window, blocking or challenging IPs that exceed configured thresholds (e.g., 2,000 requests from a single IP within 5 minutes).
*   **ElastiCache Redis Rate Limiter**: A custom distributed rate limiter where a Lambda function uses Redis atomic commands (`INCR`, `EXPIRE`) to track client limits in real time.

---

## 🔒 Idempotency & Safe Retries

In a distributed system, network timeouts can leave clients uncertain whether a write request succeeded. If a client retries, the server must avoid duplicate processing (e.g., charging a card twice).

> [!TIP]
> **Idempotent API**: An API where making multiple identical requests has the same effect as making a single request.

### Designing Idempotency on AWS
1.  **Idempotency Keys**: The client attaches a unique ID (e.g., UUID) to the request header.
2.  **State Verification**:
    ```mermaid
    sequenceDiagram
        Client->>API: POST /payments (Idempotency-Key: XYZ)
        API->>DynamoDB: Get record key "XYZ"
        alt Key exists
            DynamoDB-->>API: Return cached payment response
            API-->>Client: Return response (No duplicate charge!)
        else Key does not exist
            API->>DynamoDB: Save key "XYZ" (State: PENDING)
            API->>PaymentProcessor: Process charge
            PaymentProcessor-->>API: Success
            API->>DynamoDB: Update key "XYZ" (State: SUCCESS, Response: payload)
            API-->>Client: Return response
        end
    ```
3.  **AWS Implementation**: Use **AWS Lambda Powertools** Idempotency utility, which integrates with **Amazon DynamoDB** to enforce idempotency checks transparently.

---

## 📋 Networking & API Interview Q&A

### Question 1: How does API Gateway manage WebSocket connection states asynchronously?
**Answer**:
Under normal circumstances, maintaining persistent WebSocket TCP connections requires running a fleet of servers (like EC2/Fargate instances) that keep sockets open, consuming RAM and CPU.
**Amazon API Gateway WebSocket APIs** decouple connection management. API Gateway manages the persistent connections at its edge locations. 
*   When a client connects, API Gateway executes a Lambda function (`$connect` route) and returns a unique `connectionId`.
*   The application saves this `connectionId` to **DynamoDB**.
*   To send a message to the client, a backend service calls the API Gateway PostToConnection API, passing the `connectionId` and the payload. API Gateway locates the open socket and delivers the message.

### Question 2: Compare ALB and NLB when designing a high-throughput API gateway layer.
**Answer**:
*   **Application Load Balancer (ALB)** operates at Layer 7. It inspects HTTP headers, path routes, and cookies. It terminates TLS, allowing header manipulation (e.g., adding `X-Forwarded-For`). ALB is ideal for typical microservice REST APIs, where routing logic is based on URL paths (e.g., `/users` vs `/orders`).
*   **Network Load Balancer (NLB)** operates at Layer 4. It routes raw TCP/UDP packets, terminates TLS without inspecting contents, and does not parse HTTP. NLB scales to handle millions of requests per second with microsecond latencies. It is ideal for gaming protocols, real-time WebSockets, IoT ingestion points, and architectures requiring static or Elastic IP addresses.

### Question 3: How do you implement a distributed rate limiter that avoids the "race condition" in a multi-region deployment?
**Answer**:
In a distributed rate limiter using ElastiCache Redis, multiple parallel API requests from a single client can hit different API servers, resulting in concurrent reads and writes to Redis.
If two servers read `count = 9` simultaneously and both increment it, the database updates to `10` instead of `11`, letting a rate-limited request slip through.
To avoid this race condition:
1.  Use **Redis Transactions** (`MULTI` / `EXEC`) or **Lua Scripts**. Redis executes Lua scripts atomically, ensuring no other client can write between the read and update stages.
2.  In a multi-region deployment, synchronize rates asynchronously to avoid cross-region network latency on every API call. Use **Redis Global Datastore** to replicate rate limits asynchronously across regions, using local cache checks for critical path evaluation and global sync for background updates.

### Question 4: Users intermittently get 502s and 504s from your ALB. Walk me through troubleshooting.
**Answer**:
First separate the two codes, because they point at different failures:
*   **502 Bad Gateway**: The target *answered badly*: it closed or reset the connection, sent a malformed response, or a Lambda target returned an invalid payload. The classic cause is an app keep-alive timeout **shorter** than the ALB idle timeout (default 60s). ALB reuses a socket the target has just closed (e.g., Node.js defaults to 5s).
*   **504 Gateway Timeout**: The target *did not answer in time*. Either ALB could not open a connection to it (security group/NACL, nothing listening on the port), or the target took longer than the idle timeout (slow query, thread pool exhaustion).
*   **Where to look**: Compare `HTTPCode_ELB_5XX_Count` with `HTTPCode_Target_5XX_Count` in CloudWatch, then enable **ALB access logs** in S3. Query them with Athena for `target_status_code = '-'`, `target_processing_time` and `error_reason`.
*   **Fixes**: Set the app keep-alive timeout above the ALB idle timeout, tune slow endpoints or raise the idle timeout for them, and check `TargetResponseTime` and unhealthy host counts during deployments. Enable deregistration delay so in-flight requests drain.

### Question 5: VPC Peering vs. Transit Gateway vs. PrivateLink: how do you choose?
**Answer**:
*   **VPC Peering**: A 1:1, non-transitive link with no hourly or processing fee. It cannot have overlapping CIDRs and grows as $n(n-1)/2$ connections. It suits a handful of VPCs with high traffic volume, where TGW per-GB processing charges would add up.
*   **Transit Gateway**: A regional hub with transitive routing. It supports **route tables for segmentation** (prod cannot reach dev), VPN and Direct Connect Gateway attachments, and inter-region peering. The trade-off is a per-attachment-hour charge plus a per-GB processing charge, and one extra hop. It is the default for 10+ VPCs or any hybrid setup.
*   **PrivateLink (Interface Endpoints / Endpoint Services)**: Exposes a **single service** (behind an NLB), not a whole network. Traffic is one-way from consumer to provider, and **overlapping CIDRs are fine**. It is ideal for SaaS or a shared platform API consumed by many accounts. **VPC Lattice** extends this idea to service-to-service networking with IAM auth.
*   **Rule of thumb**: TGW for network connectivity, PrivateLink/Lattice for service exposure, peering for a few high-volume pairs.

### Question 6: Design hybrid DNS so on-prem servers resolve AWS private hosted zones and AWS workloads resolve `corp.example.com`.
**Answer**:
1.  In a central **Networking account**, create **Route 53 Resolver endpoints** in a shared-services VPC, each with ENIs in at least two AZs.
2.  **Inbound endpoint**: On-prem DNS servers get conditional forwarders for `aws.example.com`, pointing at the inbound endpoint IPs. Route 53 then answers from **Private Hosted Zones** associated with that VPC.
3.  **Outbound endpoint + forwarding rule**: Create a rule forwarding `corp.example.com` to on-prem DNS IPs over Direct Connect/VPN.
4.  **Share the rules via AWS RAM** to every workload account and associate them with their VPCs. Associate workload PHZs with the shared-services VPC (cross-account association authorization) so inbound queries can resolve them.
5.  Allow TCP/UDP 53 in the endpoint security groups, watch per-ENI query limits, and add **Route 53 Resolver DNS Firewall** to block exfiltration domains.

### Question 7: How do you plan CIDR ranges for a 200-VPC, multi-region organization that also connects to on-prem?
**Answer**:
*   Use **Amazon VPC IPAM** as the single source of truth, with hierarchical pools (e.g., `10.0.0.0/8` → a /12 per region → a /14 per environment → /20–/22 per VPC). Share pools to OUs via RAM so teams cannot pick their own ranges.
*   **Reserve the on-prem ranges** and avoid common collisions: `172.17.0.0/16` (Docker bridge) and ranges used by partners or acquired companies.
*   Size subnets for growth: AWS reserves 5 IPs per subnet. **EKS with the VPC CNI** consumes one IP per pod, so give pods a secondary CIDR from `100.64.0.0/10` with custom networking.
*   Keep routable VPC CIDRs small and summarizable so TGW/DX route tables stay under the BGP prefix limits.
*   If overlap is unavoidable (e.g., after an acquisition), expose services over **PrivateLink** or translate addresses with a **Private NAT Gateway** instead of re-IPing.
