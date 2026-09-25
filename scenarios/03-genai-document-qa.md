# Scenario 03: GenAI-Powered Document Q&A System on AWS

## 1. Problem Statement
An enterprise organization wants a secure Generative AI system that allows employees to ask natural language questions against internal corporate documents (PDFs, wikis, compliance guides). The system must restrict answer sources strictly to private files, prevent data leaks, and exclude public model training loops.

---

## 2. Requirements

### Functional
*   Allow users to submit natural language questions via a web chat interface.
*   Securely ingest, parse, and index massive corp document folders (S3).
*   Provide grounded answers referencing the exact source documents (RAG pattern).

### Non-Functional
*   **Accuracy**: Eliminate AI hallucinations by anchoring models to retrieved text context.
*   **Security**: Enforce encryption at rest for document assets and vector indexes.
*   **Performance**: Respond to user queries within 3 seconds, supporting token-by-token streaming.

---

## 3. Architecture Diagram

![GenAI Q&A and RAG Pipeline Architecture](file:///Users/hbhardwaj/Code/awesome-aws-architecture/diagrams/genai_document_qa_architecture.png)

### Interactive Mermaid Blueprint
```mermaid
graph TD
    User[Client Chat App] -->|HTTPS POST Query| APIGW[Amazon API Gateway]
    APIGW -->|Trigger Function| LambdaAgent[RAG Coordinator Lambda]
    
    subgraph Document_Ingestion_Pipeline [Document Sync Pipeline]
        DocBucket[(Enterprise Docs: S3)] -->|Object Created Notification| SQS_Ingest[(Ingestion SQS Queue)]
        SQS_Ingest --> LambdaParser[Lambda Document Chunking & Ingestion]
        LambdaParser -->|Generate Vector Embeddings| BedrockEmbed[Amazon Bedrock: Titan Text Embeddings]
        LambdaParser -->|Write Vectors| OpenSearch[(Amazon OpenSearch Serverless)]
    end
    
    subgraph Retrieval_Augmented_Generation_Path [RAG Architecture]
        LambdaAgent -->|1. Generate Query Embeddings| BedrockEmbed
        LambdaAgent -->|2. Semantic KNN Vector Search| OpenSearch
        OpenSearch -->|3. Return Relevant Text Context| LambdaAgent
        
        LambdaAgent -->|4. Structured Prompt with Context| BedrockLLM[Amazon Bedrock: Claude Sonnet]
        BedrockLLM -->|5. Token Stream Response| LambdaAgent
        LambdaAgent -->|6. Stream Response Back| APIGW
    end
```

---

## 4. Key AWS Services Used

| Service | Architectural Role | Scoped Purpose |
| :--- | :--- | :--- |
| **Amazon Bedrock** | Serverless LLM Platform. | Exposes foundation models for text generation (Claude) and embeddings generation (Titan). |
| **Amazon S3** | Raw Document Lake. | Secure, durable storage for raw PDF, DOCX, and text source files. |
| **Amazon OpenSearch Serverless**| Vector Database (Serverless). | Stores and queries vector embeddings using k-nearest neighbors (k-NN) search. |
| **AWS Lambda** | Serverless Orchestrator. | Coordinates the RAG process (retrieving context and query forwarding). |
| **Amazon API Gateway** | Web API Entry Point. | Manages HTTP/WebSocket integrations to route client chat requests. |

---

## 5. Step-by-Step Design Walkthrough
### Phase A: Document Ingestion & Vector Indexing
1.  **Storage**: Administrators upload corporate documents (PDFs, manuals) to a secure **Amazon S3 Bucket**.
2.  **Notification & Parsing**: S3 triggers an event notification to an **Amazon SQS Queue**, which invokes the **Ingestion Lambda**. The Lambda downloads the file and parses the text into smaller, overlapping chunks (e.g., 500 characters).
3.  **Embedding Creation**: For each chunk, the Lambda calls **Amazon Bedrock** using the **Titan Text Embeddings** model to convert the text into a mathematical vector representation.
4.  **Index Storage**: The vector representations are stored in **Amazon OpenSearch Serverless (AOSS)** alongside the source document metadata (file path, author, date).

### Phase B: RAG Query Execution
1.  **Client Request**: A user submits a query (e.g., *"What is our travel reimbursement policy?"*) through **API Gateway** to the **RAG Coordinator Lambda**.
2.  **Semantic Retrieval**:
    *   The Lambda converts the user query into a vector representation using the **Titan Embeddings API**.
    *   The Lambda runs a **k-Nearest Neighbors (k-NN) search** against the vector index in OpenSearch Serverless to identify the top 3 most semantically similar text chunks.
3.  **Prompt Synthesis**: The Lambda constructs a structured prompt containing the retrieved context and the user query, wrapping them inside strict system constraints:
    > *Instructions: Answer the question using ONLY the provided context. If the answer cannot be found in the context, respond with "I do not know."*
4.  **Inference**: The Lambda sends the synthesized prompt to **Amazon Bedrock (Claude Sonnet)**.
5.  **Streaming Output**: Claude generates the answer, streaming the tokens back to the Lambda. The Lambda forwards the stream to API Gateway WebSockets, providing a real-time typing experience to the client.

---

## 6. Design Patterns Applied
*   **Retrieval-Augmented Generation (RAG)**: Anchoring an LLM to external reference sources before generating a response.
*   **Semantic Search**: Searching database entries using meaning and context rather than exact keyword matching.
*   **Claim Check Pattern**: Vector indexes store lightweight text snippets and reference pointers, while S3 holds the master PDF document assets.

---

## 7. Trade-offs

### Pros
*   **Hallucination Prevention**: Restricts model responses to the provided document context, ensuring accurate answers.
*   **Serverless Efficiency**: Zero server operations or node maintenance required for Bedrock, AOSS, and Lambda.
*   **Strict Security Isolation**: Corporate data is never processed over the public internet or used to train third-party foundation models.

### Cons
*   **Embedding/Token Costs**: High API request rates can scale costs rapidly depending on the size of the retrieved context.
*   **Cold Starts**: Large python libraries (like LangChain or PyPDF) can introduce cold-start latency in Lambdas.

---

## 8. When to Use This Pattern
*   Enterprise search platforms, customer support chatbots, and internal HR wikis.
*   Any application requiring grounded, accurate natural language queries against large document datasets.

---

## 9. Cost Estimate

*   **Total Monthly Cost**: ~$500 - $1,500 (highly dependent on model usage).
*   **Key Cost Drivers**:
    *   *Amazon OpenSearch Serverless*: Ingestion & Query OCU capacity charges (~$400/month baseline fee).
    *   *Amazon Bedrock API Usage*: Bedrock bills Claude per input and output token (priced per million tokens). Output tokens cost several times more than input tokens, and costs scale with request volume.

---

## 10. Alternatives Considered & Why Rejected
*   **Fine-tuning Claude with corporate docs**: Rejected. Fine-tuning modifies internal weights but cannot guarantee accuracy or prevent hallucinations. Additionally, updating data requires running expensive training pipelines repeatedly.
*   **Use self-hosted OpenSearch on EC2**: Rejected. High administrative burden. Managing clusters, index shards, and scaling policies manually violates the Operational Excellence pillar.

---

## 11. Failure Modes & Mitigations

### 1. Document Format Parsing Failures
*   **Effect**: Ingestion pipeline crashes on scanned images or complex tables.
*   **Mitigation**: Integrate **Amazon Textract** to extract layout structures, tabular data, and scanned text accurately before running embedding pipelines.

### 2. Context Window Exhaustion
*   **Effect**: Retrieving too many text chunks exceeds the model input capacity, triggering API failures.
*   **Mitigation**: Optimize text chunk sizes (e.g., 512 tokens with 10% overlap) and set a strict limit on the number of retrieved context blocks returned by the vector query.

---

## 12. SA Interview Questions

### Question 1: How does Amazon Bedrock ensure private data isolation?
**Answer**: 
Amazon Bedrock prioritizes enterprise data security:
*   Your documents, prompt templates, and vector embeddings are stored inside your secure AWS account.
*   API calls to foundation models are fully isolated. The model provider (e.g., Anthropic) never receives your prompts or outputs.
*   None of your private corporate data is used to train the base public foundation models, preventing data leakage.

### Question 2: Why do we use OpenSearch Serverless instead of standard relational SQL databases for document search?
**Answer**: 
Relational databases rely on exact keyword matches (e.g., matching "travel policy"). If a user asks, *"Can I get refunded for my flight ticket?"*, a standard SQL query will return zero matches because the word "refunded" does not exist in the "travel policy" text. 
OpenSearch Serverless supports **Vector Embeddings and k-NN Search**, converting words into semantic vector maps. This allows identifying matching text chunks based on meaning (e.g., connecting "refunded flight" semantically to "travel reimbursement") to provide accurate search results.

---

## 🔁 Interviewer Follow-Up Drills

Real interviews push past the first design. Practice defending it against these follow-ups.

### Follow-Up 1: Chat usage grows 10x after company-wide rollout. What breaks first, and how do you fix it?
**Answer**: 
*   **First to break**: **Amazon Bedrock on-demand quotas** for Claude Sonnet (requests per minute and tokens per minute). Users see throttling errors mid-stream long before Lambda or OpenSearch Serverless run out of capacity.
*   **Fixes**:
    1.  Use **cross-region inference profiles** to spread load across regions, request quota increases, or buy **Provisioned Throughput** for a predictable baseline.
    2.  Add a **semantic cache** (e.g., ElastiCache or DynamoDB keyed by query-embedding similarity) so repeated questions like "travel policy" skip the LLM.
    3.  Set **API Gateway** usage plans and per-user throttling, plus **reserved concurrency** on the RAG Coordinator Lambda, because streaming responses hold invocations open for several seconds.
    4.  On the ingestion side, cap the SQS-triggered parser Lambda's concurrency so bulk uploads don't exhaust the **Titan Embeddings** quota shared with live queries.

### Follow-Up 2: Cut the monthly cost by 40%. What do you change, and what do you give up?
**Answer**: 
*   **Model tiering**: Route simple lookup questions to a smaller, cheaper model (e.g., Claude Haiku) and keep Sonnet for multi-document reasoning. *Give up*: Some answer quality on questions that get misrouted.
*   **Fewer tokens per request**: Tighten chunk size and the top-k retrieved chunks, and use **Bedrock prompt caching** for the fixed system prompt. *Give up*: Recall on questions whose answers span many chunks.
*   **Ingestion**: Run bulk re-embedding with **Bedrock batch inference** at a discount instead of on-demand calls.
*   **Vector store**: The **OpenSearch Serverless** OCU baseline (~$400/month) dominates at low traffic. Cap max OCUs, turn off redundant replicas for dev collections, or move low-QPS workloads to **Aurora PostgreSQL pgvector** or **S3 Vectors**. *Give up*: Higher latency and less headroom for hybrid search.

### Follow-Up 3: You need to switch embedding models (e.g., Titan v1 to Titan v2) with zero downtime. How?
**Answer**: 
1.  Vectors from different embedding models aren't comparable, so this is a **blue/green index migration**, not an in-place update.
2.  Create a new **OpenSearch Serverless index** sized for the new model's vector dimensions.
3.  Re-embed the full corpus from **S3**, which remains the source of truth (claim check pattern), by replaying an S3 Inventory listing into the ingestion SQS queue. Meanwhile, **dual-write** new uploads to both indexes.
4.  Run an offline **retrieval evaluation set** (question to expected source document) against both indexes, and compare recall and answer quality.
5.  Flip the RAG Coordinator Lambda to the new index and model through configuration (**SSM Parameter Store** or a weighted **Lambda alias** for a canary). Keep the old index for fast rollback, then delete it.

### Follow-Up 4: The OpenSearch Serverless vector collection becomes unavailable or its index is corrupted. What's the blast radius, and how do you recover?
**Answer**: 
*   **Blast radius**: Retrieval fails for every query. The critical rule is to **fail closed**. The coordinator Lambda must not call Claude without context, or the system produces ungrounded answers that break the anti-hallucination requirement. Users get a clear "search is temporarily unavailable" response.
*   **Resilience**: AOSS runs across multiple AZs with redundant replicas, so AZ failure is handled by the service. Most real incidents are bad writes, mapping errors, or accidental deletes.
*   **Recovery**: The index is **derived data**. Rebuild it from S3 by replaying documents through the ingestion pipeline (SQS with a **DLQ** for poison files). Keep **S3 Versioning** on the document bucket so a rebuild matches a known-good corpus.
*   **RTO driver**: Re-embedding throughput, which is limited by Titan quotas. Pre-compute and store embeddings in S3 alongside chunks so a rebuild only re-indexes and doesn't re-embed.

### Follow-Up 5: Claude Sonnet has a regional outage or sustained throttling. How does the system stay up?
**Answer**: 
*   Call models through the **Bedrock Converse API**, which is model-agnostic, so swapping models is a configuration change rather than a code rewrite.
*   Implement a **fallback chain** in the coordinator Lambda. First use a **cross-region inference profile** for the same model, then fall back to an alternative model (e.g., a smaller Claude model or another Bedrock provider). Use a **circuit breaker** so the Lambda stops hammering a failing endpoint.
*   Keep a **prompt and eval suite per fallback model**. Grounding instructions such as "answer ONLY from context" must be re-validated because different models follow them differently.
*   Apply the same **Bedrock Guardrails** configuration to every model in the chain so safety and PII filtering don't weaken during failover.
*   Tell users when a fallback model is serving their request, and alarm on the fallback rate in CloudWatch.
