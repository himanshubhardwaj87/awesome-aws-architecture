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
    -- KEYS[1] = "rl:{tenant}:{epochSecond}" (fixed 1-second window per key)
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local current = redis.call("INCR", key)
    if current == 1 then
        -- Set the TTL only when the window's key is created
        redis.call("EXPIRE", key, 2)
    end
    if current > limit then
        return 0
    end
    return 1
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

---

## 🎤 Interview Questions

### Question 1: A single short link goes viral and gets 1M redirects/sec. How does your URL shortener handle the hot key?
**Answer**:
*   **Absorb it at the edge**: Return a `302` with `Cache-Control: max-age` so **CloudFront** serves most hits without reaching the origin. A `301` is cached by browsers forever, which is cheaper but loses click analytics and makes link edits impossible.
*   **Protect Redis**: A single key lives on one shard. Add read replicas, plus a small in-process LRU cache inside the Lambda execution environment with a few seconds of TTL.
*   **Protect DynamoDB**: Adaptive capacity helps, but put **DAX** in front of reads for hot items.
*   **Keep analytics off the hot path**: Send click events asynchronously to **Kinesis Data Firehose → S3**.

### Question 2: How does your WhatsApp design guarantee per-chat ordering and delivery to users who are offline?
**Answer**:
*   A standard SQS queue does not preserve order, so give each message a **per-chat sequence number**. Assign it with a DynamoDB atomic counter on `chatId`, or use SQS FIFO with `MessageGroupId = chatId`.
*   Persist every message to DynamoDB (`chatId`, `seq`) **before** attempting delivery. The history table is the source of truth, and the WebSocket push is best effort.
*   If the Redis session lookup finds no `connectionId`, or `PostToConnection` returns `410 Gone`, mark the message undelivered and send a push notification via SNS/Pinpoint.
*   On reconnect, the client sends its last acknowledged `seq` per chat and the server replays the gap. Clients dedupe by `messageId`, which makes retries safe.

### Question 3: A tenant sending steady traffic well under its per-second limit still gets bursts of 429s. The limiter's Lua script runs `INCRBY key 1` then `EXPIRE key 1` on every allowed request, using a key of `rl:{tenant}`. Walk me through it.
**Answer**:
*   The script calls `EXPIRE key 1` on **every** allowed request. Under continuous traffic the TTL keeps getting pushed forward, so the counter never resets. It counts *all requests since the tenant was last idle for 1s*, not requests per second.
*   The count therefore creeps up until it crosses `limit`. Every request is then rejected, and rejected requests don't refresh the TTL. The key finally expires about 1s later and the cycle repeats. The result is periodic 429 storms for a compliant tenant.
*   **Fix**: Set the expiry only when the key is created (`INCR` returns 1). Better, embed the window in the key, `rl:{tenant}:{epochSecond}`, so each window is a fresh key.
*   For smoother limits, use a sliding window counter or a token bucket that stores `tokens` and `lastRefill` in a hash.
*   Verify with `TTL`/`GET` on the tenant's key and per-tenant 429 metrics in CloudWatch.

### Question 4: Your Uber-style proximity service melts in Manhattan at rush hour. How do you fix the hotspot?
**Answer**:
*   With fixed-size geohash cells, dense areas put millions of updates onto a few keys, and therefore onto one Redis shard.
*   **Adaptive cells**: Use a quadtree-like split, or a finer geohash precision in dense areas, so each cell holds a bounded number of drivers.
*   **Shard by region**: Use separate ElastiCache clusters per metro, and pick the cluster from the rider's coarse location.
*   **Reduce write volume**: Only send a location update when a driver moves more than N meters. Batch updates through **Kinesis** before writing to Redis.
*   Query the rider's cell plus neighboring cells, and cap the result count. Exact nearest-neighbor is unnecessary for matching.

### Question 5: A scheduled job occasionally runs twice in your distributed scheduler. What's going on and how do you fix it?
**Answer**:
*   **Likely causes**:
    *   The job ran longer than the SQS **visibility timeout**, so a second worker received the message.
    *   Or the DynamoDB lock lease expired while the first worker was paused (GC, throttling), and a second worker acquired it.
*   **Fixes**:
    *   Extend visibility with `ChangeMessageVisibility` heartbeats while the job runs.
    *   Renew the lease periodically, and use a **fencing token** that the job's side-effecting writes must present.
    *   Make the job idempotent. Write a `jobId#scheduledTime` completion record with a conditional put before side effects, or check it before starting.
*   For long or multi-step jobs, move execution to **Step Functions**, which gives exactly-once workflow execution per unique execution name (Standard workflows).

### Question 6: How do you stop a marketing blast from delaying OTP messages, and avoid spamming a user with duplicates?
**Answer**:
*   **Isolation**: Keep separate SQS queues, separate Lambda **event source mapping maximum concurrency**, and separate SES configuration sets / SNS origination numbers for transactional vs. marketing traffic. Then a 50M-message campaign cannot starve OTPs.
*   **Provider limits**: Throttle the marketing workers to the SES sending rate and SMS throughput quotas, rather than letting throttling errors pile up in the DLQ.
*   **Dedup**: Build an idempotency key from `userId + templateId + eventId` and store it in DynamoDB with a TTL.
*   **Per-user limits**: Enforce frequency caps (e.g., max 3 marketing messages/day) and honor opt-out and preference data before enqueueing.
