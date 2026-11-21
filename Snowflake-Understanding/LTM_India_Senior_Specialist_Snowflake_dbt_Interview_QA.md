# LTM (LTIMindtree) India — Senior Specialist: Data Engineering (Snowflake & dbt)
**Comprehensive Interview Preparation Guide (All Rounds: Q&A)**

---

## Interview Process Overview at LTM (LTIMindtree)
For a **Senior Specialist - Data Engineering** role in India, LTM typically conducts **3 rounds**:
1. **Round 1 (L1 Technical Screening):** Hands-on SQL/Python coding, core Snowflake architecture, and fundamental dbt mechanics (45–60 mins).
2. **Round 2 (L2 Deep-Dive / Client Technical Round):** Performance tuning, cost optimization, production dbt best practices, CDC patterns, and end-to-end architecture scenarios (45–60 mins).
3. **Round 3 (Techno-Managerial / Delivery Round):** Project ownership, stakeholder management, production incident handling, Agile/CI-CD workflows, and behavioral fit (30–45 mins).

---

## Round 1: L1 Technical Screening (Core Snowflake, dbt & Live SQL)

### Q1. Explain Snowflake's Multi-Cluster Shared Data Architecture.
**Answer:**
Snowflake separates storage, compute, and cloud services into three independent layers:
* **Database Storage Layer:** Stores structured and semi-structured data in compressed, columnar micro-partitions (50 MB–500 MB uncompressed) on cloud object storage (AWS S3, Azure Blob, or GCS).
* **Query Processing (Compute) Layer:** Uses independent Virtual Warehouses (MPP compute clusters) that read from shared storage without competing for resources.
* **Cloud Services Layer:** Coordinates authentication, metadata management, query parsing/optimization, access control, and transaction management.

---

### Q2. What is the difference between `COPY INTO` and `Snowpipe`?
**Answer:**
| Feature | `COPY INTO` | `Snowpipe` |
| :--- | :--- | :--- |
| **Execution Mode** | Batch / scheduled bulk loading | Continuous micro-batch ingestion |
| **Compute Used** | User-managed Virtual Warehouse | Snowflake-managed serverless compute |
| **Trigger** | Manual SQL, Snowflake Task, or external orchestrator (Airflow/ADF) | Cloud event notifications (S3 Event/Event Grid) or REST API |
| **Latency** | Minutes to hours | Near real-time (seconds to a couple of minutes) |

---

### Q3. How do Zero-Copy Cloning and Time Travel work together, and how do they affect storage costs?
**Answer:**
* **Time Travel** lets you query, clone, or restore historical data (`AT` or `BEFORE` a timestamp/query ID) within a retention period (1 day standard, up to 90 days in Enterprise Edition).
* **Zero-Copy Cloning** creates a metadata-only copy of a database, schema, or table pointing to the exact same underlying micro-partitions—incurring **zero additional storage cost** initially.
* **Cost Impact:** Storage costs only increase when rows are inserted, updated, or deleted in either the source or the clone, because Snowflake writes new micro-partitions for the modified data while retaining old ones for Time Travel/Fail-safe.

---

### Q4. Explain the four primary dbt materializations and when to use each.
**Answer:**
1. **`view` (Default):** Rebuilds as a SQL view on every run. Best for lightweight transformations and staging layers where freshness matters and query volume is low.
2. **`table`:** Drops and recreates the table (`CREATE OR REPLACE TABLE`) on each run. Best for medium-sized marts queried frequently by BI tools.
3. **`incremental`:** Inserts or merges only new/updated records since the last run using `is_incremental()`. Best for large fact/event tables where full refreshes are too slow or expensive.
4. **`ephemeral`:** Not built in the database; injected as a Common Table Expression (CTE) into downstream models. Best for intermediate helper logic that shouldn't clutter the warehouse schema.

---

### Q5. How does `is_incremental()` work in dbt, and what are the incremental strategies on Snowflake?
**Answer:**
`is_incremental()` evaluates to `True` only when all four conditions are met:
1. The model is configured with `materialized='incremental'`.
2. The target table already exists in Snowflake.
3. The `--full-refresh` flag was **not** passed.
4. The model is running in a standard `dbt run` context.

**Snowflake Incremental Strategies:**
* **`merge` (Default when `unique_key` is set):** Performs a SQL `MERGE` to update existing rows and insert new ones.
* **`delete+insert`:** Deletes rows matching the `unique_key` (or incremental predicate) and re-inserts them—often faster than `merge` when updating entire date partitions.
* **`append`:** Blindly inserts new rows without checking for duplicates (fastest when source data is strictly immutable).

---

### Q6. Live SQL Coding: Deduplicate a table keeping the latest record per `customer_id`.
**Answer:**
```sql
SELECT *
FROM raw_customers
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY customer_id 
    ORDER BY updated_at DESC
) = 1;
```
*Note: Mentioning `QUALIFY` demonstrates Snowflake-specific fluency compared to wrapping a window function in a CTE.*

---

### Q7. Live SQL Coding: Flatten a nested JSON array stored in a `VARIANT` column.
**Answer:**
Suppose `orders_raw` has a `payload` (`VARIANT`) column containing `{"order_id": 101, "items": [{"sku": "A1", "qty": 2}, {"sku": "B2", "qty": 1}]}`:
```sql
SELECT
    payload:order_id::INT AS order_id,
    f.value:sku::STRING AS item_sku,
    f.value:qty::INT AS quantity
FROM orders_raw,
LATERAL FLATTEN(input => payload:items) f;
```

---

## Round 2: L2 Deep-Dive / Client Architecture Round

### Q8. A Snowflake query is running slowly. How do you diagnose and fix it using the Query Profile?
**Answer:**
1. **Spilling to Local / Remote Storage:** Indicates the warehouse lacks sufficient memory for large sorts, aggregations, or joins. *Fix:* Scale up the warehouse size (e.g., Medium to Large) or optimize the query to reduce intermediate row volume.
2. **Poor Partition Pruning (Partitions Scanned ≈ Partitions Total):** Snowflake is scanning almost the entire table. *Fix:* Filter on clustered columns, avoid wrapping filter columns in functions (e.g., `DATE(ts) = ...` prevents pruning; use `ts >= ... AND ts < ...`), or define a Clustering Key on large multi-terabyte tables.
3. **Exploding Joins:** Output rows vastly exceed input rows due to many-to-many join keys. *Fix:* Deduplicate join keys prior to joining or verify join predicates.
4. **Queuing Overload:** Query waited before execution started. *Fix:* Enable multi-cluster auto-scaling or separate heavy batch ETL workloads from BI users onto dedicated warehouses.

---

### Q9. When do you Scale Up vs. Scale Out a Snowflake Virtual Warehouse?
**Answer:**
* **Scale Up (Vertical — e.g., Small -> Large):** Doubles compute nodes and memory per query. Use this for **complex, heavy queries**, large joins, or when seeing **disk spilling**.
* **Scale Out (Horizontal — Multi-Cluster Warehouse, e.g., Min 1 to Max 4 clusters):** Adds identical clusters to handle **high concurrency** (many BI users or parallel dbt models running simultaneously) when queries are queuing.

---

### Q10. Compare Snowflake Dynamic Tables vs. Streams + Tasks for building CDC pipelines.
**Answer:**
* **Streams + Tasks:** Imperative approach. A Stream tracks DML changes (inserts/updates/deletes) via offsets, and a scheduled Task executes custom SQL (`MERGE`) to apply changes. Best when you need **complex custom business logic**, multi-target branching, or event-driven triggers.
* **Dynamic Tables:** Declarative approach. You define the target state via a `SELECT` query and specify a `TARGET_LAG` (e.g., `5 minutes`). Snowflake automatically manages the incremental refresh pipeline and dependency DAG. Best for **streamlined declarative transformations** without managing custom `MERGE` boilerplate.

---

### Q11. How do you implement Slowly Changing Dimensions (SCD Type 2) in dbt?
**Answer:**
Using **dbt Snapshots**, which automatically manage `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, and `dbt_updated_at`:
* **`timestamp` strategy (Recommended):** Relies on a reliable `updated_at` column in the source table to detect changed rows.
* **`check` strategy:** Compares a list of specified columns (or `check_cols='all'`) between the source and snapshot table when no reliable `updated_at` timestamp exists.
* *Tip:* Always snapshot raw source tables (before staging transformations) so historical changes are captured accurately even if downstream business logic evolves.

---

### Q12. How do you optimize CI/CD pipelines in dbt for large enterprise projects?
**Answer:**
Use **Slim CI** powered by dbt state comparison:
1. Store the production `manifest.json` artifact from the latest `main` branch deployment.
2. During a Pull Request check, run:
   ```bash
   dbt build --select state:modified+ --defer --state ./prod_artifacts
   ```
3. **`state:modified+`** executes only models changed in the PR and their downstream dependents.
4. **`--defer`** resolves unmodified upstream `ref()` pointers directly to production tables in Snowflake instead of rebuilding the entire upstream DAG in the PR schema.

---

### Q13. How do you handle PII data governance in Snowflake?
**Answer:**
* **Object Tagging:** Classify columns with tags (e.g., `PII_TYPE = 'EMAIL'`, `'SSN'`).
* **Dynamic Data Masking:** Attach tag-based or column-level Masking Policies that check `CURRENT_ROLE()` or `IS_ROLE_IN_SESSION()`—returning unmasked values to authorized roles and masked values (e.g., `'***-**-XXXX'`) to analysts.
* **Row-Access Policies:** Restrict row visibility dynamically based on user role or mapping tables (e.g., regional managers only see rows for their assigned country).

---

### Q14. Scenario: How would you design a Medallion Architecture using AWS S3/Azure Blob, Snowflake, and dbt?
**Answer:**
1. **Landing / Ingestion:** Raw files (JSON, Parquet, CSV) land in cloud storage (S3/ADLS). **Snowpipe** (for streaming/micro-batch) or scheduled **`COPY INTO`** loads data into a **Bronze (`RAW`)** database in Snowflake with minimal transformation (retaining raw `VARIANT` payloads and ingestion metadata timestamps).
2. **Staging & Cleaning (dbt `staging` layer):** Views over Bronze tables that rename columns to snake_case, cast data types, flatten JSON arrays, and run source freshness + basic schema tests (`not_null`, `unique`).
3. **Silver (`INTERMEDIATE` / `INTEGRATED` layer):** Incremental dbt models and Snapshots that deduplicate records, apply business rules, standardize dimensions, and track SCD Type 2 history.
4. **Gold (`MARTS` / `ANALYTICS` layer):** Business-aligned dimensional models (Star Schema facts and dimensions) materialized as tables/incremental models, secured with Row-Access/Masking policies, and served to Power BI / Tableau.

---

## Round 3: Techno-Managerial & Delivery Round

### Q15. How do you control and monitor Snowflake costs in a production project?
**Answer:**
* **Resource Monitors:** Set credit quotas at account and warehouse levels with automated `NOTIFY`, `SUSPEND`, and `SUSPEND_IMMEDIATE` thresholds.
* **Auto-Suspend & Auto-Resume:** Configure aggressive `AUTO_SUSPEND` (e.g., 60 seconds for ETL warehouses; 5 minutes for BI warehouses to preserve local cache) with `AUTO_RESUME = TRUE`.
* **Workload Isolation:** Separate warehouses for ingestion, dbt transformations, BI reporting, and ad-hoc data science so costs are attributable and workloads don't block each other.
* **Query & Storage Audits:** Monitor `SNOWFLAKE.ACCOUNT_USAGE` views (`QUERY_HISTORY`, `WAREHOUSE_METERING_HISTORY`, `TABLE_STORAGE_METRICS`) to catch runaway queries and stale temporary tables.

---

### Q16. A production dbt model failed overnight due to unexpected duplicate keys from an upstream source. How do you handle it?
**Answer:**
1. **Immediate Triage & Containment:** Check the dbt run logs and Snowflake `QUERY_HISTORY`. Verify if `severity: error` on the `unique` test halted downstream mart builds so corrupted data didn't reach BI dashboards.
2. **Root Cause Analysis:** Query the raw landing table to determine whether the duplicate came from a re-ingested file (pipeline retry) or an upstream application bug.
3. **Short-Term Fix:** Quarantine or deduplicate the bad batch in the staging/incremental logic, re-run the failed model (`dbt build --select +failed_model+`), and verify row counts.
4. **Long-Term Prevention:** Add a `dbt source freshness` check, enforce deduplication (`QUALIFY ROW_NUMBER()`) in the staging layer, and set up automated Slack/Teams alerts via dbt Cloud or Airflow.

---

### Q17. How do you safely backfill 2 years of historical data in a large dbt incremental model?
**Answer:**
Running a single `--full-refresh` on 2 years of data can cause memory spills and massive credit spikes. Instead:
1. Pass custom date range variables (`--vars '{"start_date": "2024-01-01", "end_date": "2024-03-31"}'`) into the incremental filter block.
2. Temporarily scale up the dedicated transformation warehouse (e.g., from Medium to XL) for higher throughput.
3. Run the backfill in **quarterly or monthly chunks** sequentially, validating row counts and credits after each batch, then scale the warehouse back down.

---

### Q18. What questions should you ask the LTM panel at the end of the interview?
**Answer:**
* *"Is this role aligned to a greenfield cloud migration to Snowflake, or optimizing and scaling an existing dbt + Snowflake platform?"*
* *"What orchestration tool is the client using alongside dbt (e.g., dbt Cloud, Airflow, Azure Data Factory, or Dagster)?"*
* *"What are the biggest data volume or latency challenges the team is currently tackling?"*
