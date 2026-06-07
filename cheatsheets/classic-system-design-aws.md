# Classic System Design Interview Questions on AWS

A cheat sheet mapping 10 classic system design interview problems to AWS-native architectures. Each scenario details the scale, core trade-offs, failure modes, and contains a Mermaid diagram.

---

## 1. Design a URL Shortener (TinyURL)

### Architecture Diagram
```mermaid
graph TD
    Client[Client Browser] -->|1. GET /xyz| CloudFront[Amazon CloudFront CDN]
    CloudFront -->|Cache Miss| APIGW[API Gateway - Redirects]
    APIGW -->|Trigger| LambdaRedirect[Lambda Redirector]
    LambdaRedirect -->|Query Cache| Redis[(ElastiCache Redis)]
    LambdaRedirect -->|Fallback DB Query| DynamoDB[(Amazon DynamoDB)]
    
    Client -->|2. POST /shorten| APIGW_Write[API Gateway - Write]
    APIGW_Write -->|Trigger| LambdaShorten[Lambda Shortener]
    LambdaShorten -->|Base62 ID Generation| DynamoDB
```

### Architectural Details
*   **Scale**: 100M URLs created/day; 10B redirects/day (115,000 read requests/sec).
*   **Storage**: 100M URLs $\times$ 500 bytes/URL = 50 GB/day. Use DynamoDB partition scaling.
*   **Mechanism**: Base62 encoding ($[a-z, A-Z, 0-9]$) of an auto-incrementing ID or hash value. A 7-character string supports $62^7 \approx 3.5 \text{ Trillion}$ URLs.
*   **Cache Strategy**: CloudFront caches popular short URL redirects globally. Local read cache in **ElastiCache Redis** handles hot redirects (90% read cache hit ratio target).
*   **Failure Modes**: Cold cache hits database limits. Mitigation: Use DynamoDB read autoscaling and overprovision Redis.

---

## 2. Design WhatsApp (Real-time Messaging)

### Architecture Diagram
```mermaid
graph TD
    ClientA[Client A] -->|1. WebSocket Conn| APIGW[API Gateway WebSocket API]
    APIGW -->|2. Route Conn Event| LambdaConnect[Lambda Connection Mgr]
    LambdaConnect -->|3. Register Session| SessionDB[(ElastiCache Redis - Sessions)]
    
    ClientA -->|4. Send Message| APIGW
    APIGW -->|5. Publish Msg| MessageQueue[(Amazon SQS Queue)]
    MessageQueue -->|6. Process Msg| LambdaDelivery[Lambda Message Router]
    LambdaDelivery -->|7. Look Up Recipient| SessionDB
    LambdaDelivery -->|8. Push to Client B| APIGW
    APIGW -->|9. Deliver WebSocket Msg| ClientB[Client B]
    
    LambdaDelivery -->|10. Backup History| HistoryDB[(Amazon DynamoDB)]
```

### Architectural Details
*   **Scale**: 1B active users/day, 100B messages/day (1.15M messages/sec).
*   **Mechanism**: Persistent WebSockets established via API Gateway WebSocket API. API Gateway manages the persistent connections at its edge, calling Lambda dynamically.
*   **Presence & Session Store**: ElastiCache Redis stores mapping of `userId -> connectionId` and online status.
*   **History & Offline Storage**: DynamoDB partition key: `chatId`, sort key: `messageId`.
*   **Failure Modes**: Connection stampede during network reconnects. Mitigation: Backoff and jitter on client reconnect logic; scale API Gateway endpoints.

---

## 3. Design Spotify / Video Streaming

### Architecture Diagram
```mermaid
graph TD
    User[Client Application] -->|1. Stream Request| CloudFront[Amazon CloudFront CDN]
    CloudFront -->|Segment Hits| User
    
    Creator[Media Creator] -->|2. Upload Media| S3_Raw[(Amazon S3 - Raw Input)]
    S3_Raw -->|S3 Event Notification| StepTranscode[AWS Step Functions Transcoder]
    StepTranscode -->|3. Trigger Job| Transcoder[AWS Elemental MediaConvert]
    Transcoder -->|4. Split HLS Segments| S3_Segments[(Amazon S3 - Segment Store)]
    S3_Segments -->|Origin Egress| CloudFront
```

### Architectural Details
*   **Scale**: 100M active listeners, 5M songs played concurrently.
*   **Mechanism**: Audio/Video is split into small segments (usually 6-second fragments) and encoded into different bitrates using HTTP Live Streaming (HLS) or Dynamic Adaptive Streaming over HTTP (DASH).
*   **Storage & CDN**: Source raw media is saved in S3. **AWS Elemental MediaConvert** encodes media into HLS profiles. **CloudFront** caches segments near users.
*   **Failure Modes**: Mid-stream lag due to network drops. Mitigation: Adaptive Bitrate Streaming (ABR) automatically switches clients to lower quality profiles dynamically.

---

## 4. Design Uber / Yelp (Proximity Service)

### Architecture Diagram
```mermaid
graph TD
    Driver[Driver App] -->|1. Coordinates / 4s| APIGW_WS[API Gateway WebSockets]
    APIGW_WS -->|Route Coordinates| DriverLambda[Lambda Locator]
    DriverLambda -->|2. Update Location| GeoCache[(ElastiCache Redis - Geo)]
    
    Rider[Rider App] -->|3. Search Drivers| APIGW_HTTP[API Gateway REST]
    APIGW_HTTP -->|Trigger Query| RiderLambda[Lambda Search]
    RiderLambda -->|4. Geo Radius Query| GeoCache
    RiderLambda -->|5. Historical Log| DynamoDB[(Amazon DynamoDB)]
```

### Architectural Details
*   **Scale**: 10M active drivers updating locations every 4 seconds (2.5M updates/sec).
*   **Mechanism**: Geo-hashing (dividing the physical map into grid zones). ELastiCache Redis natively supports Geospatial indexes (`GEOADD`, `GEORADIUS`) using sorted sets (ZSETs).
*   **Matching Flow**: Riders query surrounding drivers within a geohash prefix. Real-time driver paths are synchronized via API Gateway WebSocket routes.
*   **Failure Modes**: Hotspots in dense urban areas (e.g., Manhattan). Mitigation: Dynamic partition splitting of Redis Geospatial clusters or scaling grids dynamically (QuadTree algorithms).

---

## 5. Design a Distributed Rate Limiter

### Architecture Diagram
```mermaid
graph TD
    Client[Client App] -->|1. API Call| CloudFront[Amazon CloudFront CDN]
    CloudFront -->|2. Forward Request| APIGW[Amazon API Gateway]
    APIGW -->|3. Authorize & Limit| CustomAuth[Lambda Authorizer]
    CustomAuth -->|4. Check Window Counter| Redis[(ElastiCache Redis)]
    
    alt Under Limit
        CustomAuth -->|Allow| BackendLambda[Lambda Application Service]
    else Rate Limited
        CustomAuth -->|Deny - 429 Too Many Requests| Client
    end
```

### Architectural Details
*   **Scale**: 1M API requests/sec across a multi-tenant client base. Low latency overhead (< 5ms).
*   **Mechanism**: Token Bucket or Sliding Window Counter.
*   **Redis Lua Scripting**: The Lambda Authorizer runs an atomic Lua script on Redis:
    ```lua
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local current = tonumber(redis.call('get', key) or "0")
    if current + 1 > limit then
        return 0
    else
        redis.call("INCRBY", key, 1)
        redis.call("EXPIRE", key, 1)
        return 1
    end
    ```
*   **Failure Modes**: Redis availability outage blocks all API traffic. Mitigation: Fail-open fallback (if Redis queries time out, allow requests but trigger alerts).

---

## 6. Design Google Docs (Collaborative Editing)

### Architecture Diagram
```mermaid
graph TD
    ClientA[Client A] -->|1. WebSocket Connection| APIGW[API Gateway WebSockets]
    APIGW -->|Route Edit Op| ECS_Fargate[ECS Fargate - Sync Service]
    
    ECS_Fargate -->|2. Read/Write Doc State| Redis[(ElastiCache Redis)]
    ECS_Fargate -->|3. Resolve Conflicts OT / CRDT| ECS_Fargate
    ECS_Fargate -->|4. Broadcast Changes| APIGW
    APIGW -->|5. Deliver Update| ClientB[Client B]
    
    ECS_Fargate -->|6. Asynchronous Save| DocStore[(Amazon DocumentDB / S3)]
```

### Architectural Details
*   **Scale**: 10M active documents, 1M users editing concurrently.
*   **Mechanism**: Conflict resolution using Operational Transformation (OT) or Conflict-Free Replicated Data Types (CRDTs).
*   **Server Cluster**: Persistent WebSockets route editing operations to ECS Fargate task containers. The containers run the collaboration engine, caching documents in ElastiCache Redis.
*   **Failure Modes**: Connection drops cause local edits to diverge. Mitigation: Client buffers operations and performs delta updates upon reconnection.

---

## 7. Design a Unique ID Generator (Snowflake ID)

### Architecture Diagram
```mermaid
graph TD
    Client[Client App] -->|Request Unique ID| APIGW[Amazon API Gateway]
    APIGW -->|Route Request| LambdaGen[Lambda ID Generator]
    
    subgraph Snowflake ID Layout (64-Bits)
        SignBit[1-Bit Sign]
        Timestamp[41-Bits Millisecond Epoch]
        MachineID[10-Bits Worker ID]
        Sequence[12-Bits Counter]
    end
```

### Architectural Details
*   **Scale**: Generate 100,000 unique, chronologically ordered IDs per second.
*   **Layout**:
    *   *Sign bit*: 1 bit.
    *   *Timestamp*: 41 bits (gives 69 years of millisecond resolution).
    *   *Machine/Worker ID*: 10 bits (supports 1,024 concurrent worker nodes).
    *   *Sequence number*: 12 bits (supports 4,096 unique IDs per millisecond per node).
*   **Implementation**: Lambdas retrieve their unique Worker ID dynamically from ECS metadata or DynamoDB lease registers.
*   **Failure Modes**: Clock drift (system time goes backward). Mitigation: Reject ID generation requests if local server clock is behind the last logged timestamp.

---

## 8. Design a Web Crawler

### Architecture Diagram
```mermaid
graph TD
    Scheduler[Crawl Scheduler] -->|1. Enqueue Seed URLs| URL_Queue[(Amazon SQS URL Queue)]
    URL_Queue -->|2. Poll Target URL| Crawler[ECS Crawler Worker]
    
    Crawler -->|3. Fetch HTML| WebServer[Target Web Server]
    Crawler -->|4. Extract Links| LinkExtractor[ECS Link Extractor]
    
    LinkExtractor -->|5. Filter Visited| RedisBloom[(ElastiCache Redis - Bloom Filter)]
    LinkExtractor -->|6. Enqueue New Link| URL_Queue
    
    Crawler -->|7. Store HTML Page| RawS3[(Amazon S3 Content Store)]
```

### Architectural Details
*   **Scale**: Crawl 5B web pages per month.
*   **Mechanism**: SQS holds target URL queues. ECS Crawler Workers fetch pages, respecting `robots.txt` rate limits (politeness policies).
*   **Deduplication**: ElastiCache Redis stores a **Bloom Filter** to determine if a URL has already been visited, avoiding infinite crawl loops.
*   **Storage**: S3 stores parsed HTML content, compressed via Gzip.
*   **Failure Modes**: Trapped in spider traps (infinite generated pages). Mitigation: Limit crawl depth per host domain.

---

## 9. Design a Distributed Job Scheduler

### Architecture Diagram
```mermaid
graph TD
    EventBridge[Amazon EventBridge Scheduler] -->|1. Trigger Execution| JobQueue[(Amazon SQS Job Queue)]
    JobQueue -->|2. Consume Job| LambdaWorker[Lambda Execution Worker]
    
    LambdaWorker -->|3. Acquire Lock| LockTable[(Amazon DynamoDB Lock Table)]
    LambdaWorker -->|4. Execute Job Task| LambdaWorker
    LambdaWorker -->|5. Log Output & Status| StatusTable[(Amazon DynamoDB Job Status)]
```

### Architectural Details
*   **Scale**: Execute 10M scheduled jobs/day; guarantee at-least-once or exactly-once delivery.
*   **Mechanism**: **Amazon EventBridge Scheduler** scales to trigger cron or one-time jobs, forwarding payloads to SQS.
*   **Concurrence & Safety**: DynamoDB serves as the lease locking table. Worker threads acquire a lock on a target job before executing to guarantee a job isn't processed by multiple workers concurrently.
*   **Failure Modes**: Worker crashes during job execution. Mitigation: Configure SQS visibility timeouts to return failed jobs back to the queue automatically if a heartbeat is not updated.

---

## 10. Design a Scalable Notification Service

### Architecture Diagram
```mermaid
graph TD
    Client[Client Platform] -->|1. Trigger Notification| APIGW[Amazon API Gateway]
    APIGW -->|Route| LambdaIngest[Lambda Ingestion]
    
    LambdaIngest -->|2. High Priority| QueueHigh[(SQS Priority Queue)]
    LambdaIngest -->|2. Low Priority| QueueLow[(SQS Marketing Queue)]
    
    QueueHigh -->|3. Pull| SenderHigh[Lambda Delivery Worker]
    QueueLow -->|3. Pull| SenderLow[Lambda Delivery Worker]
    
    SenderHigh -->|4. Send Email| SES[Amazon SES]
    SenderHigh -->|4. Send SMS| SNS[Amazon SNS]
    SenderHigh -->|4. Send Push| Pinpoint[Amazon Pinpoint]
```

### Architectural Details
*   **Scale**: Send 1B notifications/day (Email, SMS, Push alerts).
*   **Mechanism**: Decouple ingest from delivery. Incoming payloads are classified into Priority Queues (e.g., SQS High Priority for OTP/2FA, SQS Low Priority for Marketing).
*   **AWS Delivery Integrations**: SNS handles SMS and push alerts, SES handles transactional emails, and Amazon Pinpoint handles campaign targeting.
*   **Failure Modes**: Downstream provider throttling (e.g., carrier SMS block). Mitigation: Configure SQS Dead Letter Queues (DLQs) to retry failed notifications automatically with exponential backoff.
