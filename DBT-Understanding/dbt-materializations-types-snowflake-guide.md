# dbt Materializations & Snowflake Governance Guide

A quick-reference cheat sheet covering the four primary dbt materializations, Snowflake SQL examples, and production governance best practices.

---

## 1. Quick Comparison Matrix

| Materialization | Snowflake Object Created | Data Persistence | Query Speed (Read) | Build Speed (Write) | Best Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **View** | `VIEW` | None (SQL logic only) | Slower (runs live) | Instant | Lightweight staging & renaming |
| **Table** | `TRANSIENT` or `PERMANENT TABLE` | Full physical copy | Fast | Moderate to Slow | BI marts, heavy joins & aggregations |
| **Incremental** | `TABLE` (merged/appended) | Physical (delta updates) | Fast | Fast (after 1st run) | Large event logs, CDC, high-volume fact tables |
| **Ephemeral** | None (injected as CTE) | None | Depends on downstream | None | Reusable intermediate snippets |

---

## 2. Snowflake Examples

### A. View (`materialized='view'`)
Use in your **staging (`stg_`) layer** to cast types, rename columns, and filter soft-deleted records without duplicating raw storage.

```sql
-- models/staging/stg_orders.sql
{{
    config(
        materialized='view',
        secure=false  -- Set to true if hiding view definition from non-owners
    )
}}

SELECT
    id AS order_id,
    user_id AS customer_id,
    order_date::DATE AS order_date,
    status AS order_status,
    _etl_loaded_at
FROM {{ source('raw_ecommerce', 'orders') }}
WHERE status != 'test_order'
```

---

### B. Table (`materialized='table'`)
Use in your **marts (`dim_` / `fct_`) layer** where BI tools (Tableau, Power BI, Sigma) query repeatedly and need fast response times.

```sql
-- models/marts/dim_customers.sql
{{
    config(
        materialized='table',
        transient=false,                -- Retain Fail-safe for critical prod marts
        cluster_by=['signup_month']     -- Optional: Only for multi-TB tables
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
)

SELECT
    c.customer_id,
    c.email,
    DATE_TRUNC('month', c.created_at) AS signup_month,
    COUNT(o.order_id) AS lifetime_orders,
    COALESCE(SUM(o.order_amount), 0) AS lifetime_revenue
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY 1, 2, 3
```

---

### C. Incremental (`materialized='incremental'`)
Use for **large fact or event tables** (millions/billions of rows) where rebuilding from scratch wastes Snowflake warehouse credits.

```sql
-- models/marts/fct_events.sql
{{
    config(
        materialized='incremental',
        unique_key='event_id',
        incremental_strategy='merge',   -- Snowflake supports: merge, append, delete+insert, microbatch
        on_schema_change='sync_all_columns'
    )
}}

SELECT
    event_id,
    user_id,
    event_name,
    event_timestamp,
    updated_at
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
    -- 3-day lookback window handles late-arriving data safely
    WHERE updated_at >= (
        SELECT DATEADD('day', -3, MAX(updated_at))
        FROM {{ this }}
    )
{% endif %}
```

> **Note on Full Refreshes:** Run `dbt run --full-refresh --select fct_events` whenever you change historical business logic or alter column calculations.

---

### D. Ephemeral (`materialized='ephemeral'`)
Use for **simple, intermediate transformations** that shouldn't clutter Snowflake schemas with extra objects.

```sql
-- models/intermediate/int_active_subscriptions.sql
{{
    config(
        materialized='ephemeral'
    )
}}

SELECT
    subscription_id,
    customer_id,
    plan_tier
FROM {{ ref('stg_subscriptions') }}
WHERE status = 'active'
  AND current_period_end > CURRENT_TIMESTAMP()
```

When referenced via `{{ ref('int_active_subscriptions') }}`, dbt compiles it directly into the downstream query as a CTE:

```sql
-- Compiled Snowflake SQL in downstream model
WITH __dbt__cte__int_active_subscriptions AS (
    SELECT
        subscription_id,
        customer_id,
        plan_tier
    FROM analytics.staging.stg_subscriptions
    WHERE status = 'active'
      AND current_period_end > CURRENT_TIMESTAMP()
)
SELECT * FROM __dbt__cte__int_active_subscriptions
```

---

## 3. Governance & Snowflake Best Practices

### Project-Level Defaults (`dbt_project.yml`)
Enforce materializations by folder rather than configuring every file manually.

```yaml
models:
  my_analytics_project:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: marts
      +transient: false
```

### Snowflake Cost & Storage Governance
1. **Use `transient=true` for Staging & Dev Tables:** Snowflake `TRANSIENT` tables skip the 7-day Fail-safe storage cost while keeping Time Travel (0–1 day). Set `+transient: true` globally in `dbt_project.yml` and override `+transient: false` only for critical production marts.
2. **Right-Size Warehouses per Layer:** Route heavy incremental merges to a larger warehouse (e.g., `MEDIUM` or `LARGE`) using `snowflake_warehouse` config, while keeping views and small tables on `X-SMALL`.
3. **Avoid Deep View Chains:** Never stack more than 2–3 views on top of each other. If a view takes >10 seconds to query, promote it to a `table`.
4. **Always Set `on_schema_change`:** On incremental models, configure `on_schema_change='sync_all_columns'` or `'append_new_columns'` so upstream schema additions don't silently fail or drop columns.
5. **Enforce Contracts & Tests:** Add `unique` and `not_null` tests on `unique_key` columns in incremental models—duplicate keys will cause Snowflake `MERGE` statements to fail with a nondeterministic row error.
6. **Protect Production Incremental Tables:** Set `full_refresh: false` in the config block of massive multi-terabyte incremental models to prevent accidental `dbt run --full-refresh` commands from wiping history.
