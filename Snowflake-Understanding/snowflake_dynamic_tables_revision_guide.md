# Snowflake Dynamic Tables: Senior Engineer Interview Revision Guide

> Checked against the Snowflake docs on 2026-11-23. Dynamic tables change fast; items marked 🆕 are newer (2025–2026) and may be missing from older blogs or an interviewer's notes.

## 1. The 30-Second Answer

> "A dynamic table is a table defined by a `SELECT` query plus a **target lag**. Snowflake materializes the result and keeps it fresh automatically: it tracks changes on the base tables, works out the dependency graph, and runs incremental or full refreshes on a warehouse I choose. I declare **what** the table should contain and **how stale** it may be; Snowflake handles scheduling and orchestration. It replaces most stream + task + `MERGE` pipelines with one statement."

Four things to remember:

| Concept | One line |
|---|---|
| **Query** | The `SELECT` that defines the contents. |
| **Target lag** | Maximum staleness you accept (minimum 60 seconds). A goal, not a guarantee. |
| **Refresh mode** | How it is refreshed: incremental (changed rows only) or full (recompute everything). |
| **Warehouse** | The compute that runs the refreshes. You pay for it. |

```
                 change tracking                refresh (incremental/full)
 raw_orders ─────────────────────► dt_orders_clean ─────────────────► dt_customer_revenue
 (base table)                      (dynamic table)                    (dynamic table)
 dim_customers ──────────────────────────────────────────────────────────────┘
```

---

## 2. Running Example (used in the whole guide)

```sql
CREATE OR REPLACE TABLE raw_orders (
  order_id INT, customer_id INT, order_ts TIMESTAMP_NTZ, amount NUMBER(10,2), status STRING
);
INSERT INTO raw_orders VALUES
  (1001, 1, '2026-11-20 09:00', 500, 'COMPLETED'),
  (1002, 1, '2026-11-20 10:00', 300, 'COMPLETED'),
  (1003, 2, '2026-11-20 11:00', 700, 'COMPLETED'),
  (1004, 2, '2026-11-21 09:30', 200, 'CANCELLED');

CREATE OR REPLACE TABLE dim_customers (customer_id INT, customer_name STRING, region STRING);
INSERT INTO dim_customers VALUES (1, 'Asha', 'South'), (2, 'Ravi', 'North');
```

### Step 1: A cleaning layer (silver)

```sql
CREATE OR REPLACE DYNAMIC TABLE dt_orders_clean
  TARGET_LAG   = DOWNSTREAM          -- refresh only when a table that reads from me needs it
  WAREHOUSE    = transform_wh
  REFRESH_MODE = INCREMENTAL
AS
SELECT order_id, customer_id, order_ts::DATE AS order_date, amount
FROM raw_orders
WHERE status = 'COMPLETED';
```

### Step 2: An aggregate layer (gold) that reads the first dynamic table

```sql
CREATE OR REPLACE DYNAMIC TABLE dt_customer_revenue
  TARGET_LAG   = '10 minutes'        -- at most 10 minutes behind raw_orders / dim_customers
  WAREHOUSE    = transform_wh
  REFRESH_MODE = INCREMENTAL
AS
SELECT c.customer_id, c.customer_name, c.region,
       COUNT(*)      AS order_count,
       SUM(o.amount) AS total_revenue
FROM dt_orders_clean o
JOIN dim_customers  c ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name, c.region;
```

No stream, no task, no `MERGE`, no "run A before B". Snowflake reads the two queries and builds the graph `raw_orders → dt_orders_clean → dt_customer_revenue` by itself.

**Right after creation** (order 1004 is CANCELLED, so it is filtered out):

| customer_id | customer_name | region | order_count | total_revenue |
|---|---|---|---|---|
| 1 | Asha | South | 2 | 800 |
| 2 | Ravi | North | 1 | 700 |

### Step 3: Change the base tables and watch the result

You only touch the **base tables**. Each change below shows what `dt_customer_revenue` looks like after the next refresh.

```sql
-- (a) INSERT a new completed order for Ravi
INSERT INTO raw_orders VALUES (1005, 2, '2026-11-21 12:00', 400, 'COMPLETED');
--   Ravi:  order_count 1 → 2,  total_revenue 700 → 1100

-- (b) UPDATE an amount
UPDATE raw_orders SET amount = 550 WHERE order_id = 1001;
--   Asha:  total_revenue 800 → 850

-- (c) DELETE an order
DELETE FROM raw_orders WHERE order_id = 1003;
--   Ravi:  order_count 2 → 1,  total_revenue 1100 → 400

-- (d) A row starts matching the WHERE filter
UPDATE raw_orders SET status = 'COMPLETED' WHERE order_id = 1004;
--   Row 1004 now appears in dt_orders_clean
--   Ravi:  order_count 1 → 2,  total_revenue 400 → 600

-- (e) Change the OTHER side of the join
UPDATE dim_customers SET region = 'West' WHERE customer_id = 1;
--   Asha:  region South → West   (every table in the join is tracked)
```

**Final result:**

| customer_id | customer_name | region | order_count | total_revenue |
|---|---|---|---|---|
| 1 | Asha | West | 2 | 850 |
| 2 | Ravi | North | 2 | 600 |

Key points this example proves:

* Inserts, updates **and deletes** flow through. You write no merge logic.
* With incremental refresh, change (a) only recomputes **Ravi's** group. Asha's row is not touched.
* You cannot write to the dynamic table yourself:

```sql
INSERT INTO dt_orders_clean VALUES (9999, 1, '2026-11-21', 10);   -- Error: it is maintained by Snowflake
-- Only exception 🆕: rows inside a "frozen region" (section 9) accept DML.
```

---

## 3. Target Lag (most-asked concept)

**Target lag = how stale the data may be. It is NOT "refresh every N minutes".**

```
TARGET_LAG = '10 minutes'

10:00  Row inserted into raw_orders
10:06  Snowflake starts a refresh. It reads the base tables "as of 10:06" (the data timestamp).
10:07  Refresh commits. The row is visible. Staleness of that row was 7 minutes: within target.
```

* Snowflake chooses the refresh times. It usually refreshes **more often** than the lag, because the refresh's own run time eats into the budget.
* It is **best effort**. If a refresh takes 25 minutes, a 10-minute lag cannot be met and the data is simply stale. Nothing errors.
* A new refresh never overlaps a running one for the same table. The scheduled attempt is **skipped** instead (see section 8).
* Minimum is **60 seconds**. Lag is measured against the **root base tables**, not the dynamic table directly above.

### `TARGET_LAG = DOWNSTREAM`

"I have no schedule of my own. Refresh me only when a dynamic table that reads from me needs fresh data." Use it for intermediate (staging / silver) tables.

| `dt_orders_clean` (upstream) | `dt_customer_revenue` (downstream) | What happens |
|---|---|---|
| `DOWNSTREAM` | `'10 minutes'` | ✅ **Recommended.** Gold stays within 10 minutes; silver refreshes on demand. |
| `'5 minutes'` | `'10 minutes'` | Works. Silver also refreshes on its own schedule (extra cost if nobody reads silver directly). |
| `'10 minutes'` | `DOWNSTREAM` | ❌ Gold **never refreshes**: nothing reads from it. No warning is raised. |
| `DOWNSTREAM` | `DOWNSTREAM` | ❌ Neither refreshes automatically. Manual refresh only. |
| `'1 hour'` | `'10 minutes'` | ❌ Error. A table cannot be fresher than its upstream. |

```sql
ALTER DYNAMIC TABLE dt_customer_revenue SET TARGET_LAG = '1 hour';   -- cheap change, no rebuild
ALTER DYNAMIC TABLE dt_orders_clean     SET TARGET_LAG = DOWNSTREAM;
```

---

## 4. Refresh Modes

| Mode | What it does | Use when |
|---|---|---|
| `INCREMENTAL` | Computes only the rows that changed since the last refresh and merges them in. | Small share of data changes per refresh (rule of thumb: under 5%). |
| `FULL` | Re-runs the whole query and replaces the contents. | Query uses constructs incremental cannot handle, or most of the data changes every time (truncate-and-reload). |
| `ADAPTIVE` 🆕 | Incremental normally, but rebuilds from scratch when a huge upstream change (for example `INSERT OVERWRITE`) makes incremental more expensive. | Snowflake's recommended choice for incremental workloads. |
| `AUTO` (default) | Snowflake picks **once, at creation**. It then stays fixed. | Quick experiments. Avoid in production. |
| `CUSTOM_INCREMENTAL` 🆕 | You write the refresh DML yourself (`REFRESH USING (...)`). | Advanced cases the built-in logic cannot express. |

Facts interviewers probe:

* `AUTO` resolves to `ADAPTIVE` or `FULL` today. Older material (and tables created earlier) say `INCREMENTAL` or `FULL`. Both answers show you know it is decided **once at creation and never re-evaluated**.
* If `AUTO` picks `FULL`, the `CREATE` still **succeeds silently**. You find out from the cost. Always check.
* If you write `REFRESH_MODE = INCREMENTAL` and the query is not supported, the `CREATE` **fails with an error**. That is why production code sets the mode explicitly.
* Switching `FULL ↔ INCREMENTAL` rebuilds the table. Switching `INCREMENTAL ↔ ADAPTIVE` does not.

```sql
-- Always verify what you actually got
SHOW DYNAMIC TABLES LIKE 'dt_%' ->> SELECT "name", "refresh_mode", "refresh_mode_reason" FROM $1;
-- name                 | refresh_mode | refresh_mode_reason
-- DT_CUSTOMER_REVENUE  | INCREMENTAL  | NULL
-- DT_TOP_ORDERS        | FULL         | Unsupported query operator type ...   ← tells you WHY it is not incremental
```

### Why can some queries not be incremental?

Incremental works when the new rows can be processed without re-reading the old ones.

* `SUM(amount)` per customer: 5 new orders arrive → add them to the affected customers' totals. ✅
* Median order amount: 5 new orders can move the middle value → all rows must be re-sorted. ❌ Needs `FULL`.

### What blocks incremental refresh

| Construct | Incremental? | Typical fix |
|---|---|---|
| `INNER` / `CROSS JOIN`, `UNION ALL`, `UNION`, `GROUP BY`, `DISTINCT`, CTEs, most window functions, `LATERAL FLATTEN`, `QUALIFY` | ✅ Yes | |
| Outer joins with `=` predicates | ✅ Yes | |
| Outer joins with non-equality predicates (`ON a.id > b.id`) | ❌ No | Restructure so the join key is an equality, or use `FULL`. |
| Subqueries outside `FROM` (`WHERE EXISTS`, `WHERE x IN (SELECT ...)`) | ❌ No | Rewrite as a `JOIN`. |
| `EXCEPT` / `MINUS` / `INTERSECT` | ❌ No | `EXCEPT` → `LEFT JOIN ... WHERE b.key IS NULL`. `INTERSECT` → inner join + `DISTINCT`. |
| `LIMIT` / `TOP` | ❌ No | `QUALIFY ROW_NUMBER() OVER (...) <= n`. |
| `GROUP BY ROLLUP` / `CUBE` / `GROUPING SETS` | ❌ No | Several `GROUP BY` queries with `UNION ALL`. |
| `WITH RECURSIVE`, `CONNECT BY`, `UNPIVOT`, sequences | ❌ No | Use `FULL`. |
| Exact percentiles (`PERCENTILE_CONT`, `MEDIAN`) | ❌ No | `APPROX_PERCENTILE` 🆕, or `FULL`. |
| `CURRENT_TIMESTAMP` / `CURRENT_DATE` in the `SELECT` list | ❌ No | Allowed only in `WHERE` / `HAVING` / `QUALIFY`. |
| External functions, `SAMPLE` | ❌ Not allowed in **any** mode | Use streams + tasks. |

**Before → after example:**

```sql
-- ❌ Falls back to FULL (subquery in WHERE)
SELECT o.* FROM raw_orders o
WHERE EXISTS (SELECT 1 FROM dim_customers c WHERE c.customer_id = o.customer_id);

-- ✅ Incremental (same result, written as a join; customer_id is unique in dim_customers)
SELECT o.* FROM raw_orders o
JOIN dim_customers c ON c.customer_id = o.customer_id;
```

```sql
-- ❌ Falls back to FULL (LIMIT): top 10 orders by amount
SELECT * FROM raw_orders ORDER BY amount DESC LIMIT 10;

-- ✅ Incremental: same top 10, written with a window function
SELECT * FROM raw_orders
QUALIFY ROW_NUMBER() OVER (ORDER BY amount DESC) <= 10;

-- ✅ Incremental and cheaper: "latest order per customer" (PARTITION BY keeps the recomputed slice small)
SELECT * FROM raw_orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_ts DESC) = 1;
```

`ORDER BY` in the definition is accepted but has no effect: a dynamic table has no guaranteed row order.

---

## 5. How It Works Under the Hood

1. **Change tracking** on the base tables tells Snowflake which rows changed (the same mechanism streams use).
   * Snowflake switches it on automatically at creation **if your role owns the base table**. Otherwise the owner must run:
     ```sql
     ALTER TABLE raw_orders SET CHANGE_TRACKING = TRUE;
     ```
2. A background scheduler checks for changes (this is **cloud services** compute).
3. If there are changes, the warehouse runs the refresh. If nothing changed, the warehouse is **not** started.
4. The result is applied **atomically**: readers see the old version or the new one, never half a refresh.

### Data timestamp and snapshot isolation

Every refresh has a **data timestamp**. The table then equals "the query result as of that moment".

All dynamic tables in one pipeline are refreshed for the **same** data timestamp:

```
Refresh cycle, data timestamp = 10:06:00
  dt_orders_clean      reads raw_orders       as of 10:06:00
  dt_customer_revenue  reads dt_orders_clean  as of 10:06:00
                       reads dim_customers    as of 10:06:00   ← same instant
```

Without this, gold could join "orders from 10:06" with "customers from 09:50" and produce wrong numbers.

**Failures cause staleness, never inconsistency.** If `dt_orders_clean` fails, `dt_customer_revenue` is not refreshed (state `UPSTREAM_FAILED`) and keeps its last good version.

### Initialization

| Setting | Behaviour |
|---|---|
| `INITIALIZE = ON_CREATE` (default) | `CREATE` blocks until the first full load finishes. A bad query fails immediately. |
| `INITIALIZE = ON_SCHEDULE` | `CREATE` returns at once. The table is **empty** until the first scheduled refresh. |

`INITIALIZATION_WAREHOUSE = big_wh` 🆕 lets the heavy first load (and later rebuilds) use a larger warehouse than the routine refreshes.

### Reinitialization (a forced full rebuild) happens when

* You run `CREATE OR REPLACE` on the dynamic table itself.
* An **upstream** table, view, dynamic table or UDF is recreated with `CREATE OR REPLACE`, even with identical columns.
* A column used by the query is dropped and re-added on a base table.
* A masking or row access policy is added to or removed from a base table (incremental tables).
* The refresh mode changes between `FULL` and `INCREMENTAL` / `ADAPTIVE`.

These do **not** rebuild: changing target lag or warehouse, suspend / resume, adding a clustering key, adding an unrelated column to a base table.

---

## 6. Dynamic Tables vs Streams + Tasks vs Materialized Views vs Views

| | Dynamic table | Stream + task | Materialized view | View |
|---|---|---|---|---|
| Style | Declarative (`SELECT`) | Imperative (you write the `MERGE`) | Declarative | Declarative |
| Stores data | Yes | Yes (your target table) | Yes | No |
| Joins / multi-table | ✅ | ✅ | ❌ single table only | ✅ |
| Freshness | Target lag, ≥ 1 minute | Whatever schedule you set | Always current | Always current |
| Scheduling | Automatic from lag | You define (CRON, task graph) | Automatic | None |
| Compute | **Your warehouse** | Your warehouse or serverless | Serverless (Snowflake-managed) | Query time |
| Optimizer rewrites queries to use it | ❌ | ❌ | ✅ | ❌ |
| Procedural logic, stored procedures, external calls | ❌ | ✅ | ❌ | ❌ |
| Can chain into a pipeline | ✅ native | ✅ manual | ❌ | ✅ |
| Edition | All | All | Enterprise+ | All |

**Rule of thumb for the interview:**

* Logic fits in a `SELECT` and a minute or more of lag is fine → **dynamic table**.
* Need procedural steps, custom `MERGE`, external calls, exact CRON timing, or writes to several targets → **streams + tasks**.
* Speed up repeated queries on **one** table, always current, transparently → **materialized view**.

### Same pipeline, both ways

**Streams + tasks (4 objects, hand-written merge logic):**

```sql
CREATE TABLE orders_clean (order_id INT, customer_id INT, order_date DATE, amount NUMBER(10,2));
CREATE STREAM raw_orders_stream ON TABLE raw_orders;

CREATE TASK load_orders_clean
  WAREHOUSE = transform_wh
  SCHEDULE  = '10 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('raw_orders_stream')
AS
MERGE INTO orders_clean t
USING (
  SELECT * FROM raw_orders_stream
  WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE)      -- drop the "old half" of an update
) s
ON t.order_id = s.order_id
WHEN MATCHED AND s.METADATA$ACTION = 'DELETE'      THEN DELETE
WHEN MATCHED AND s.status <> 'COMPLETED'           THEN DELETE      -- no longer passes the filter
WHEN MATCHED                                       THEN UPDATE SET t.customer_id = s.customer_id,
                                                                   t.order_date  = s.order_ts::DATE,
                                                                   t.amount      = s.amount
WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' AND s.status = 'COMPLETED'
                                                   THEN INSERT VALUES (s.order_id, s.customer_id, s.order_ts::DATE, s.amount);

ALTER TASK load_orders_clean RESUME;                                -- tasks are created suspended
```

**Dynamic table (1 object):**

```sql
CREATE DYNAMIC TABLE dt_orders_clean
  TARGET_LAG = '10 minutes'  WAREHOUSE = transform_wh
AS
SELECT order_id, customer_id, order_ts::DATE AS order_date, amount
FROM raw_orders WHERE status = 'COMPLETED';
```

---

## 7. Managing Dynamic Tables: Command Cheat Sheet

```sql
ALTER DYNAMIC TABLE dt_customer_revenue REFRESH;                    -- refresh now (upstream dynamic tables too)
ALTER DYNAMIC TABLE dt_customer_revenue SUSPEND;                    -- stop scheduled refreshes; data stays queryable
ALTER DYNAMIC TABLE dt_customer_revenue RESUME;
ALTER DYNAMIC TABLE dt_customer_revenue SET TARGET_LAG = '30 minutes';
ALTER DYNAMIC TABLE dt_customer_revenue SET WAREHOUSE  = transform_wh_l;
ALTER DYNAMIC TABLE dt_customer_revenue CLUSTER BY (customer_id);

SHOW DYNAMIC TABLES IN SCHEMA mydb.myschema;
DESCRIBE DYNAMIC TABLE dt_customer_revenue;
SELECT GET_DDL('DYNAMIC_TABLE', 'dt_customer_revenue');

DROP   DYNAMIC TABLE dt_customer_revenue;
UNDROP DYNAMIC TABLE dt_customer_revenue;
CREATE DYNAMIC TABLE dt_customer_revenue_dev CLONE dt_customer_revenue;
```

**Changing the query:** you cannot `ALTER` the columns or the `SELECT`.

| Statement | Effect |
|---|---|
| `CREATE OR REPLACE DYNAMIC TABLE ...` | Drops and recreates. **Always** rebuilds. Grants are lost unless you add `COPY GRANTS`. |
| `CREATE OR ALTER DYNAMIC TABLE ...` 🆕 | Changes the existing object in place. Rebuilds only if the query or refresh mode changed. Good for CI/CD. |

### Suspend behaviour

* Suspending a table also suspends **everything downstream** of it.
* Resuming it automatically resumes the tables that were suspended by that cascade. A table you suspended **by hand** must be resumed by hand.
* **Auto-suspend:** after **5 consecutive failed scheduled refreshes**, Snowflake suspends the table and its downstream tables. It does **not** resume by itself after you fix the problem; run `RESUME`.
* **Trap:** if an incremental table stays suspended longer than the base table's Time Travel retention (default 1 day), the change data is gone. It cannot resume; you must recreate it.

### `SUSPEND` vs `SCHEDULER = DISABLE` 🆕

| | `SUSPEND` | `SET SCHEDULER = DISABLE` |
|---|---|---|
| Meaning | Temporary pause | Snowflake never schedules it |
| Target lag | Kept | Reported as `NULL` |
| Who triggers refresh | Nobody until `RESUME` | An external orchestrator (Airflow, dbt) running `ALTER ... REFRESH` |
| Re-enable | `RESUME` | Set a new `TARGET_LAG` |

---

## 8. Monitoring and Troubleshooting

```sql
-- 1. Are my tables meeting their lag?
SELECT name, scheduling_state, last_completed_refresh_state,
       target_lag_sec, maximum_lag_sec, time_within_target_lag_ratio
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES());
-- time_within_target_lag_ratio = 0.62  → on target only 62% of the time: bigger warehouse or longer lag

-- 2. What did each refresh do?
SELECT name, state, state_code, state_message, refresh_action, refresh_trigger,
       data_timestamp, refresh_start_time, refresh_end_time
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(NAME => 'mydb.myschema.dt_customer_revenue'))
ORDER BY refresh_start_time DESC
LIMIT 20;

-- 3. Why is it suspended? What does it depend on?
SELECT name, inputs, scheduling_state:state::STRING AS state,
       scheduling_state:reason_code::STRING AS reason_code
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_GRAPH_HISTORY())
ORDER BY valid_from DESC;
```

Snowsight also shows the pipeline as a graph, with refresh history for each table.

**Reading the refresh history:**

| Column | Value | Meaning |
|---|---|---|
| `refresh_action` | `NO_DATA` | Nothing changed upstream. Warehouse was **not** used. No compute cost. |
| | `INCREMENTAL` | Only changes were processed. |
| | `FULL` | Whole query recomputed. |
| | `REINITIALIZE` | Rebuilt from scratch (see section 5). |
| `state` | `SUCCEEDED` / `FAILED` | |
| | `UPSTREAM_FAILED` | This table is fine; something above it failed. Fix that one first. |
| | `SKIPPED` | The scheduled attempt was not run (previous refresh still running, or refreshes take far longer than the lag). |
| `refresh_trigger` | `SCHEDULED` / `MANUAL` / `CREATION` | |

**Reason codes when suspended:** `USER_SUSPENDED`, `UPSTREAM_USER_SUSPENDED`, `SUSPENDED_DUE_TO_ERRORS`, `UPSTREAM_SUSPENDED_DUE_TO_ERRORS`.

---

## 9. Large Tables: Frozen Regions and Backfill 🆕

Problem: `dt_orders_clean` holds 5 years of orders. A rebuild reprocesses all 5 years, although only the last few days ever change.

```sql
CREATE OR REPLACE DYNAMIC TABLE dt_orders_clean
  TARGET_LAG = '10 minutes'
  WAREHOUSE  = transform_wh
  FROZEN WHERE (order_date < CURRENT_TIMESTAMP() - INTERVAL '30 days')   -- older syntax: IMMUTABLE WHERE
AS
SELECT order_id, customer_id, order_ts::DATE AS order_date, amount
FROM raw_orders WHERE status = 'COMPLETED';
```

* Rows with `order_date` older than 30 days are **frozen**: refreshes and rebuilds skip them.
* Only the last 30 days (the **active region**) are recomputed.
* Side effect to mention: a late correction to a frozen row in `raw_orders` will **not** reach the dynamic table.
* Frozen rows accept direct DML, for example a GDPR delete: `DELETE FROM dt_orders_clean WHERE customer_id = 105;`

**Backfill**: migrate an existing table without recomputing history.

```sql
CREATE OR REPLACE DYNAMIC TABLE dt_orders_clean
  TARGET_LAG = '10 minutes'
  WAREHOUSE  = transform_wh
  FROZEN WHERE (order_date < '2026-01-01')
  BACKFILL FROM orders_clean_legacy                    -- old stream/task target; same columns, same order
AS
SELECT order_id, customer_id, order_ts::DATE AS order_date, amount
FROM raw_orders WHERE status = 'COMPLETED';
-- Rows before 2026-01-01: cloned from orders_clean_legacy (not recomputed)
-- Rows from 2026-01-01:   computed from the query
```

---

## 10. Cost

| Component | What drives it | Note |
|---|---|---|
| **Warehouse compute** | Each refresh that has work to do | Usually the largest part. No upstream change → warehouse stays suspended → no cost. |
| **Cloud services** | Change-detection checks on **every** cycle, even with no changes | Billed only if the account's daily cloud services exceed 10% of warehouse credits. 200 tables × 1-minute lag = 288,000 checks per day. |
| **Storage** | The materialized data, plus Time Travel and Fail-safe | Use `CREATE TRANSIENT DYNAMIC TABLE` to drop Fail-safe cost on rebuildable layers. |

Cost levers a senior engineer should list:

1. Loosest target lag the business accepts. Do not set 1 minute "to be safe".
2. `DOWNSTREAM` on intermediate tables.
3. Keep refreshes incremental. Check `refresh_mode` after every deployment.
4. Right-size the warehouse. Use `INITIALIZATION_WAREHOUSE` for the heavy first load.
5. Frozen regions on large historical tables.
6. Suspend non-production pipelines outside working hours.
7. Dedicated warehouse while testing, so cost is easy to attribute.

---

## 11. Privileges

```sql
-- To create
GRANT USAGE ON DATABASE mydb                         TO ROLE transform_role;
GRANT USAGE ON SCHEMA mydb.myschema                  TO ROLE transform_role;
GRANT CREATE DYNAMIC TABLE ON SCHEMA mydb.myschema   TO ROLE transform_role;
GRANT SELECT ON TABLE mydb.myschema.raw_orders       TO ROLE transform_role;
GRANT USAGE ON WAREHOUSE transform_wh                TO ROLE transform_role;

-- To use
GRANT SELECT  ON DYNAMIC TABLE dt_customer_revenue   TO ROLE analyst_role;   -- query it
GRANT MONITOR ON DYNAMIC TABLE dt_customer_revenue   TO ROLE ops_role;       -- see refresh history / state
GRANT OPERATE ON DYNAMIC TABLE dt_customer_revenue   TO ROLE ops_role;       -- suspend, resume, refresh, set lag
GRANT SELECT  ON FUTURE DYNAMIC TABLES IN SCHEMA mydb.myschema TO ROLE analyst_role;
```

* Hierarchy: `OWNERSHIP` ⊃ `OPERATE` ⊃ `MONITOR`.
* **Refreshes run as the owner role.** If that role later loses `SELECT` on a base table or `USAGE` on the warehouse, refreshes start failing, and after 5 failures the table is suspended.

---

## 12. Gotchas That Separate Senior Answers

1. **Upstream `CREATE OR REPLACE` forces a rebuild.** A dbt `table` model upstream is recreated on every run, so every dynamic table below it rebuilds on every run. Fix: make the upstream an incremental model or use `INSERT OVERWRITE` (keeps the same table object), or accept `FULL` / `ADAPTIVE`.
2. **Silent `FULL`.** `AUTO` can pick full refresh without an error. Check `refresh_mode_reason`.
3. **Target lag is not an SLA.** Monitor `time_within_target_lag_ratio`.
4. **Schema changes.** Dropping or renaming a base column used by the query makes refreshes fail. `SELECT *` in the definition lets new columns flow through automatically.
5. **Masking / row access policies** on base tables are evaluated at **refresh time as the owner role**, and that result is what gets stored. Policies using context functions (for example `CURRENT_ROLE()`) can also block incremental refresh. Fix for both: put the policy on the **dynamic table**, where it runs at query time for each reader.
6. **Zero-row refresh still costs.** Updating a column the query never reads still counts as a change, so the warehouse starts and finds nothing to write.
7. **Long suspension** past Time Travel retention breaks incremental tables (section 7).
8. **Views that hide dynamic tables.** A dynamic table reading a view that itself reads a dynamic table must wrap it in `DYNAMIC_TABLE_REFRESH_BOUNDARY()` 🆕. That input then refreshes independently and snapshot isolation does not apply across it.
9. **Not real time.** Minimum lag is 60 seconds. For lower latency, look at Snowpipe Streaming into a base table plus views, or hybrid tables.
10. **Streams on a dynamic table** work, but after a rebuild the stream returns the entire table as changes.

### With dbt

```sql
{{ config(
    materialized = 'dynamic_table',
    snowflake_warehouse = 'TRANSFORM_WH',
    target_lag = '10 minutes',
    refresh_mode = 'INCREMENTAL',
    on_configuration_change = 'apply'
) }}
SELECT order_id, customer_id, order_ts::DATE AS order_date, amount
FROM {{ source('raw', 'raw_orders') }}
WHERE status = 'COMPLETED'
```

`dbt run` only creates or alters the object. **Snowflake** refreshes it afterwards, so you no longer need a scheduled `dbt run` for freshness. A dynamic table replaces a dbt incremental model when the logic is a plain `SELECT`.

---

## 13. Scenario Questions (with model answers)

**Q1. A dynamic table has a 5-minute target lag but the dashboard is 40 minutes stale. What do you check?**
Refresh history first: look at durations and `SKIPPED` rows. If each refresh takes 30+ minutes, the lag cannot be met and attempts are skipped. Then check `refresh_action`: if it is `FULL` or `REINITIALIZE` where I expected `INCREMENTAL`, find out why (`refresh_mode_reason`, or an upstream `CREATE OR REPLACE`). Fix in this order: make it incremental, simplify or split the query, scale up the warehouse, and only then relax the lag. Also check whether the table or an upstream is suspended.

**Q2. The pipeline stopped overnight. Nothing was deployed.**
Query `DYNAMIC_TABLE_GRAPH_HISTORY` for `reason_code`. `UPSTREAM_SUSPENDED_DUE_TO_ERRORS` means walk upstream to the table showing `SUSPENDED_DUE_TO_ERRORS`, read its `state_message` in refresh history (typical causes: dropped column, revoked grant, dropped warehouse). Fix the cause, then `ALTER DYNAMIC TABLE <root> RESUME`; the cascaded downstream tables resume with it.

**Q3. Dynamic table costs doubled this month.**
Split the cost into the three components. Common causes: a table silently on `FULL` refresh; an upstream job that started doing truncate-and-reload so every refresh rewrites everything; many tables at 1-minute lag pushing cloud services over the 10% threshold; an oversized warehouse; upstream updates that produce zero-row refreshes. Join `DYNAMIC_TABLE_REFRESH_HISTORY` to `QUERY_HISTORY` on `query_id` to see which table burns the credits.

**Q4. You have a 3 TB table loaded by streams and tasks. How do you migrate to a dynamic table without recomputing history?**
Create the dynamic table with `BACKFILL FROM` the existing table and a `FROZEN WHERE` predicate covering the history. The history is cloned from the old table; only the active region is computed. Validate counts, repoint consumers, then suspend the task and drop the stream.

**Q5. When would you NOT use a dynamic table?**
Sub-minute latency; logic that needs stored procedures, loops or external function calls; a job that must run at an exact CRON time; one job writing to several targets; targets that the application must also write to; or when I need the optimizer to transparently rewrite queries (that is a materialized view).

**Q6. How do dynamic tables guarantee two downstream tables agree with each other?**
Snapshot isolation: every table in the pipeline is refreshed for the same data timestamp, upstream first, and each refresh is atomic. If something fails, downstream tables keep the previous consistent version instead of mixing old and new inputs.

**Q7. Incremental refresh is slower than full refresh on one table. Why?**
Incremental has overhead to find and merge changes. It loses when a large share of rows change per refresh, when the query stacks several `GROUP BY` / `DISTINCT` / window steps, or with heavy deletes. Options: `FULL`, `ADAPTIVE`, or split the query into several simpler dynamic tables.

---

## 14. Rapid-Fire Revision

* Definition: `SELECT` + `TARGET_LAG` + `WAREHOUSE`. Snowflake keeps it fresh.
* Target lag: staleness goal, min 60 s, best effort, measured from the root base tables.
* `DOWNSTREAM`: refresh only when a consumer needs it. Never on the last table in the chain.
* Downstream lag must be ≥ upstream lag.
* Modes: `INCREMENTAL`, `FULL`, `ADAPTIVE`, `AUTO` (decided once at creation), `CUSTOM_INCREMENTAL`.
* Explicit `INCREMENTAL` + unsupported query = error. `AUTO` + unsupported query = silent `FULL`.
* Needs change tracking on base tables. Refresh runs as the **owner role**.
* Snapshot isolation across the pipeline; failures mean stale data, not inconsistent data.
* Read-only for users (except frozen regions).
* 5 consecutive scheduled failures → auto-suspend, cascades downstream, manual `RESUME`.
* No changes upstream → `NO_DATA` → no warehouse cost (cloud services check still happens).
* Upstream `CREATE OR REPLACE` → full rebuild.
* Privileges: `CREATE DYNAMIC TABLE`, `SELECT`, `MONITOR`, `OPERATE`, `OWNERSHIP`.
* vs MV: joins allowed, your warehouse, has lag, no query rewrite.
* vs stream + task: declarative, automatic DAG, but no procedural logic.
