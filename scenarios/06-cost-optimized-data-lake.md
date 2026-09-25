# Scenario 06: Cost-Optimized Data Lake on AWS (Modern Data Stack)

## 1. Problem Statement
An enterprise organization aggregates gigabytes of raw multi-source telemetry, transactional, and logging data daily. The data platform must ingest, structure, and analyze this information securely and cost-effectively, bypassing expensive, continuous database server costs while supporting high-performance BI reporting.

---

## 2. Requirements

### Functional
*   Ingest raw, unstructured log files and database tables from diverse sources.
*   Catalog, clean, and convert raw inputs automatically into structured analytical formats.
*   Allow data analysts to run fast ad-hoc SQL queries against the raw data pool.
*   Enforce fine-grained data access controls (column and row-level masking).

### Non-Functional
*   **Cost Efficiency**: Eliminate running database compute costs when the platform is idle.
*   **Scale**: Store petabytes of structured historical records securely.
*   **Agnostic Data Access**: Support modern open-table formats (Apache Iceberg) to prevent vendor lock-in.

---

## 3. Architecture Diagram

This data stack implements a modern **Lakehouse Architecture**, utilizing serverless analytics engines to query open-table datasets stored directly on Amazon S3.

![Cost-Optimized Data Lake on AWS Architecture](file:///Users/hbhardwaj/Code/awesome-aws-architecture/diagrams/cost_optimized_data_lake_architecture.png)

### Interactive Mermaid Blueprint
```mermaid
graph TD
    Sources[Log Streams, Databases, IoT Telemetry] -->|Raw Files Ingest| Firehose[Amazon Data Firehose]
    
    subgraph Data_Lake_S3_Storage [Data Lake Storage: S3 Bucket Tiers]
        Firehose -->|1. Land JSON/CSV| S3_Raw[(S3 Raw Zone - Standard Tiers)]
        
        S3_Raw -->|2. Trigger Transformation| GlueETL[AWS Glue Serverless Spark Job]
        
        GlueETL -->|3. Partition & Convert to Apache Iceberg| S3_Analytics[(S3 Analytics Zone - Intelligent-Tiering)]
        S3_Analytics -.->|Lifecycle Rules| S3_Cold[(S3 Deep Archive - Cold Backups)]
    end
    
    subgraph Metadata_Governance [Data Governance & Catalog]
        GlueETL -->|4. Register Tables| Catalog[AWS Glue Data Catalog]
        Catalog <-->|5. Row & Column Access Management| LakeFormation[AWS Lake Formation]
    end
    
    subgraph Query_Engines_Layer [Serverless Query Engine Layer]
        LakeFormation <--> Athena[Amazon Athena Serverless SQL]
        LakeFormation <--> Spectrum[Amazon Redshift Spectrum]
        
        Athena -->|Ad-Hoc Analysts Queries| AnalyticsUsers[Data Analyst / BI Tools: QuickSight]
        Spectrum -->|Enterprise Warehouse Joins| WarehouseUsers[Data Warehouse Reports]
    end
```

---

## 4. Key AWS Services Used

| Service | Architectural Role | Scoped Purpose |
| :--- | :--- | :--- |
| **Amazon S3** | Object Data Lake. | Serves as the primary storage layer, leveraging intelligent-tiering to optimize costs. |
| **Amazon Data Firehose**| Ingestion Stream Engine. | Ingests, aggregates, and flushes streaming data directly to S3. |
| **AWS Glue** | Serverless ETL. | Transforms unstructured JSON data into optimized Apache Iceberg (Parquet) formats. |
| **AWS Glue Data Catalog**| Central Metadata Registry.| Indexes database schemas and partition locations for analytical engines. |
| **AWS Lake Formation** | Data Governance Center. | Enforces fine-grained, row/column-level access control permissions. |
| **Amazon Athena** | Serverless SQL Engine. | Queries S3 tables directly using standard SQL without provisioning databases. |
| **Amazon Redshift Spectrum**| Serverless Queries. | Extends Redshift queries to scan S3 raw data directly, joining S3 and warehouse tables. |

---

## 5. Step-by-Step Design Walkthrough
1.  **Ingestion**: Multi-source logs and streaming payloads are collected by **Amazon Data Firehose**, which aggregates events and writes raw CSV/JSON files directly to the **S3 Raw Zone**.
2.  **ETL Processing**: S3 upload triggers an **AWS Glue ETL Spark Job**. The job cleanses records, removes duplicate fields, and converts data into **Apache Iceberg** table format (backing data with column-oriented **Apache Parquet** files).
3.  **Analytics Storage**: The optimized Iceberg tables are stored in the **S3 Analytics Zone** configured with **S3 Intelligent-Tiering** to optimize costs dynamically as files age.
4.  **Metadata Registration**: Glue registers the Iceberg schema and file partition updates inside the **AWS Glue Data Catalog**.
5.  **Access Management**: **AWS Lake Formation** manages security, defining who can access specific database columns and rows.
6.  **Serverless Querying**:
    *   Data analysts run ad-hoc queries against S3 directly using **Amazon Athena**, paying only for the volume of data scanned per query.
    *   Enterprise reports join historical S3 data with real-time operational database tables inside Amazon Redshift using **Redshift Spectrum**.
7.  **Archival**: S3 lifecycle policies automatically move raw, un-cleansed legacy log files from the raw zone into **S3 Glacier Deep Archive** to minimize long-term storage costs.

---

## 6. Design Patterns Applied
*   **Lakehouse Architecture**: Running serverless SQL query engines directly over file storage (S3) without importing data into expensive relational databases.
*   **Open-Table Format (Apache Iceberg)**: Supports transactional consistency (ACID), schema evolution, and high performance for multi-engine analytical environments on S3.
*   **Column-Oriented Storage Conversion**: Converting flat text formats (CSV, JSON) into compressed, columnar formats (Parquet) to optimize search speeds and lower costs.

---

## 7. Trade-offs

### Pros
*   **Exceptional Cost Efficiency**: Serverless Athena, Glue, and S3 eliminate idle server fees entirely. If no queries run, you pay strictly for storage.
*   **Massive Scalability**: S3 scales storage infinitely, while serverless query engines handle petabyte-scale datasets.
*   **Unified Governance**: Lake Formation enforces security across all analytics engines from a central console.

### Cons
*   **Query Performance Variations**: Scanning cold, unstructured S3 files is slower than querying fully indexed operational warehouses (like a native Redshift cluster).
*   **Complex Transformation Management**: Developing and maintaining Spark Glue ETL jobs requires data engineering expertise.

---

## 8. When to Use This Pattern
*   Enterprise data platforms holding massive historical datasets with variable query rates.
*   Ad-hoc analysis pipelines where maintaining running database clusters 24/7 is not cost-effective.

---

## 9. Cost Estimate

*   **Total Monthly Cost**: ~$300 - $1,500/month (varies with query volume).
*   **Key Cost Drivers**:
    *   *Amazon S3 Storage*: Billed per GB. Using Glacier and Intelligent-Tiering minimizes base costs.
    *   *Amazon Athena Queries*: $5 per TB of data scanned.
    *   *AWS Glue Spark execution fees*: Billed per DPU (Data Processing Unit) hour consumed during ETL runs.

---

## 10. Alternatives Considered & Why Rejected
*   **Store all analytical records in Amazon Redshift**: Rejected. Storing petabytes of historical logs inside a running Redshift cluster is extremely expensive, as you pay continuously for active compute nodes even when queries are not running.
*   **Use self-managed Hadoop/Presto on EC2 (EMR)**: Rejected. High operational overhead. Provisioning, scaling, and managing clusters manually violates operational excellence principles.

---

## 11. Failure Modes & Mitigations

### 1. High Athena Query Costs
*   **Effect**: Users run un-optimized queries (e.g., `SELECT *` without filters), scanning terabytes of data and generating high bills.
*   **Mitigation**: Enforce **partitioning** inside the data catalog. Ensure Athena queries use where filters to restrict searches to specific partition dates. Configure Athena **query limits** to cancel expensive queries automatically.

### 2. Schema Drift Failures
*   **Effect**: Source database updates change columns, crashing the Glue ETL ingestion jobs.
*   **Mitigation**: Configure the Glue Crawler or ETL Spark job to automatically detect schema drifts and log alerts, updating target tables dynamically without disrupting operations.

---

## 12. SA Interview Questions

### Question 1: What is the benefit of using Apache Iceberg format over basic Parquet files in an S3 Data Lake?
**Answer**: 
*   **Basic Parquet on S3** lacks a transactional management layer. You cannot run `UPDATE` or `DELETE` statements easily, schema changes (like renaming columns) require rewriting all historical files, and query engines can return inconsistent results if files are written and read simultaneously.
*   **Apache Iceberg** is an open-table format that brings relational database features directly to S3. It supports full ACID transactions, allows safe column updates and schema evolution without data rewrites, and optimizes performance via smart partition pruning, making S3 look and perform like an operational SQL database.

### Question 2: How does converting CSV files to Parquet format lower your Amazon Athena bill?
**Answer**: 
Amazon Athena bills strictly based on the volume of data scanned ($5.00 per Terabyte scanned).
1.  **Compression**: Parquet format uses advanced algorithms to compress data, reducing raw storage size by up to 80% compared to flat CSV files.
2.  **Columnar Storage**: CSV is row-oriented; to find values in a single column, Athena must scan the entire file. Parquet is column-oriented. If a query only requests the `sales_revenue` column, Athena only scans that specific column block in S3, bypassing all other columns. This can reduce data scan volumes by over 90%, lowering costs proportionally.

---

## 🔁 Interviewer Follow-Up Drills

Real interviews push past the first design. Practice defending it against these follow-ups.

### Follow-Up 1: Daily ingest volume and query load grow 10x. What breaks first, and how do you fix it?
**Answer**: 
*   **First to break**: The **per-upload Glue trigger**. Every Firehose flush starts a Glue Spark job, which hits **Glue concurrent job limits**. Concurrent jobs writing the same **Iceberg** table cause commit conflicts under optimistic concurrency, and frequent small flushes create a **small-files problem** that slows Athena and raises S3 GET costs.
*   **Fixes**:
    1.  Switch to **scheduled micro-batch** Glue runs with **job bookmarks** for incremental processing, or have **Firehose write directly to Iceberg tables**.
    2.  Increase **Firehose buffer size and interval** so it writes fewer, larger files.
    3.  Enable **Glue Data Catalog automatic compaction** for Iceberg tables, or schedule `OPTIMIZE`.
    4.  Isolate heavy BI in separate **Athena workgroups** (or Athena provisioned capacity), and push steady dashboard workloads to **Redshift Serverless** or QuickSight **SPICE**.

### Follow-Up 2: Cut the monthly data platform bill by 40%. What do you change, and what do you give up?
**Answer**: 
*   **Glue**: Run non-urgent ETL on the **Flex execution class** (spare capacity, much cheaper per DPU-hour), enable auto-scaling, and right-size workers. *Give up*: Unpredictable start times and less fresh data.
*   **Athena**:
    *   Set **per-query data-scanned limits** in workgroups and use **partition projection**.
    *   Enable **query result reuse**.
    *   Point QuickSight at **SPICE** instead of re-running the same Athena queries on every dashboard load.
*   **S3**:
    *   Expire **Iceberg snapshots** and remove **orphan files**, since old snapshots silently keep deleted data billable.
    *   Transition the raw zone to **Glacier Deep Archive** sooner.
    *   Don't count on Intelligent-Tiering for tiny objects: objects under 128 KB are never monitored (so no monitoring fee), but they also never move down a tier and stay at Frequent Access rates. Compact small files (Iceberg `rewrite_data_files`) instead.
    *   *Give up*: Shorter time travel and 12–48 hour retrieval for old raw data.

### Follow-Up 3: How do you migrate existing Hive-style Parquet tables to Apache Iceberg with zero downtime for analysts?
**Answer**: 
1.  Put a **stable name** in front of consumers: analysts and QuickSight query an **Athena view** or a Glue table name, not the physical table.
2.  Create the Iceberg table with a Spark **`snapshot`/`add_files`** procedure in Glue. This builds Iceberg metadata over the existing Parquet files without rewriting data. Alternatively, use Athena **CTAS** when you also want to repartition.
3.  **Dual-run** the Glue ETL so new data lands in both tables. Validate row counts and aggregates.
4.  Grant access on the new table with **Lake Formation tag-based access control (LF-Tags)** so row and column policies carry over automatically.
5.  **Atomically repoint the view** to the Iceberg table. Keep the old table read-only for rollback, then retire it.

### Follow-Up 4: The Glue ETL pipeline fails for a day. What's the blast radius, and how do you recover?
**Answer**: 
*   **Blast radius**: This is contained by design. **Firehose keeps landing data in the S3 Raw Zone** because ingestion is decoupled from transformation. The **Analytics Zone goes stale**, so Athena, Redshift Spectrum, and QuickSight show yesterday's numbers. Nothing is lost or corrupted, because **Iceberg commits are atomic** and partial writes are never visible.
*   **Detection**: Set **EventBridge** rules on Glue job state `FAILED` or `TIMEOUT` to alert, and add a **data-freshness metric** (max event time in the analytics table) to catch silent no-op runs.
*   **Recovery**: Fix the job and re-run. **Job bookmarks** and idempotent `MERGE INTO` upserts reprocess the backlog without duplicates.
*   **Caveat**: Replay only works while raw files are still in S3 Standard. Keep the raw-zone lifecycle transition to **Deep Archive** longer than your worst-case recovery window.

### Follow-Up 5: An upstream team renames a column and changes another column's type without warning. How does the lake handle schema evolution?
**Answer**: 
*   **Iceberg tracks columns by ID, not by name.** Renames, adds, drops, reorders, and safe **type widening** (`int` to `long`, `float` to `double`) are metadata-only changes with **no data rewrite**, and old snapshots stay readable.
*   **Incompatible changes** (e.g., `string` to `int`) aren't in-place. Add a new column, backfill it in Glue, and deprecate the old one through a view.
*   **Prevent surprises**: Register producer schemas in the **AWS Glue Schema Registry** with **BACKWARD** compatibility so breaking changes are rejected at the producer. Have the Glue job compare incoming schemas to the catalog and **quarantine** mismatched records to an error prefix instead of crashing.
*   **Governance**: Verify that **Lake Formation** column-level grants and masks still apply to renamed or new columns. A new PII column must be tagged before analysts can see it.
*   The **raw zone** keeps the original payloads, so any bad mapping can be reprocessed.
