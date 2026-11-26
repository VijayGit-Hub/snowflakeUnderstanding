# Slowly Changing Dimensions (SCD Type 2) in dbt — Complete Guide

Self-contained notes: what an SCD is, why Type 2 exists, and exactly how to build, query, test, and run it in dbt on Snowflake. Every example uses the same four customers so you can follow the data from start to finish.

> **Versions:** The main syntax is for **dbt v1.9 and later** (YAML snapshots). The older Jinja-block syntax (dbt v1.8 and earlier) is in [Section 11](#11-legacy-syntax-dbt-v18-and-earlier), because you will still meet it in existing projects and interviews.

**Contents**

1. [The 60-second interview answer](#1-the-60-second-interview-answer)
2. [What is a Slowly Changing Dimension?](#2-what-is-a-slowly-changing-dimension)
3. [SCD types compared](#3-scd-types-compared)
4. [Anatomy of an SCD Type 2 table](#4-anatomy-of-an-scd-type-2-table)
5. [dbt snapshots: how dbt does SCD Type 2](#5-dbt-snapshots-how-dbt-does-scd-type-2)
6. [Full worked example](#6-full-worked-example)
7. [Strategies: `timestamp` vs `check`](#7-strategies-timestamp-vs-check)
8. [Hard deletes](#8-hard-deletes)
9. [Other snapshot configs](#9-other-snapshot-configs)
10. [Where snapshots are built (schemas and environments)](#10-where-snapshots-are-built-schemas-and-environments)
11. [Legacy syntax (dbt v1.8 and earlier)](#11-legacy-syntax-dbt-v18-and-earlier)
12. [Building the dimension on top of the snapshot](#12-building-the-dimension-on-top-of-the-snapshot)
13. [Querying an SCD Type 2 table](#13-querying-an-scd-type-2-table)
14. [Testing](#14-testing)
15. [Running and scheduling](#15-running-and-scheduling)
16. [SCD Type 2 without snapshots](#16-scd-type-2-without-snapshots)
17. [Gotchas](#17-gotchas)
18. [Best practices](#18-best-practices)
19. [Interview questions and answers](#19-interview-questions-and-answers)
20. [Cheat sheet](#20-cheat-sheet)

---

## 1. The 60-second interview answer

> "In dbt, SCD Type 2 is implemented with **snapshots**. A snapshot is a dbt resource that points at a mutable source table and is run with `dbt snapshot`. The first run copies the source and adds metadata columns — `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, `dbt_updated_at`. On every later run dbt compares the source with the current snapshot rows. For a changed row it **closes the old version** by setting `dbt_valid_to`, and **inserts a new version** with `dbt_valid_to = NULL`. Change detection uses one of two strategies: **`timestamp`**, which compares a reliable `updated_at` column and is preferred, or **`check`**, which compares a list of columns. Deletes are handled with the `hard_deletes` config. I snapshot the raw source with `select *`, keep snapshots in their own protected schema because they cannot be rebuilt, and build a `dim_` model on top that adds a surrogate key and an `is_current` flag. Facts join to it with a point-in-time join: `order_ts >= valid_from AND order_ts < valid_to`."

Everything below explains each part of that answer.

---

## 2. What is a Slowly Changing Dimension?

**Fact vs dimension**

| | Fact table | Dimension table |
| :--- | :--- | :--- |
| Holds | Events / measurements | Descriptive context |
| Example | `orders` (order_id, amount, order_ts) | `customers` (name, city, loyalty tier) |
| Changes | Rows are appended | Existing rows get **updated** |

A dimension is "slowly changing" because its attributes change occasionally and unpredictably: a customer moves city, a product changes category, an employee changes department.

**The problem**

Source systems usually overwrite. Asha lives in Pune, then moves to Bengaluru:

```text
Before:  customer_id=101, city='Pune'
After:   customer_id=101, city='Bengaluru'     <- 'Pune' is gone forever
```

Asha placed order 5001 (₹1,000) while living in Pune and order 5002 (₹2,500) after moving to Bengaluru. Revenue by city:

| City | If history is overwritten | If history is kept (correct) |
| :--- | ---: | ---: |
| Pune | 0 | 1,000 |
| Bengaluru | 3,500 | 2,500 |

Without history, last quarter's Pune report changes after the fact. SCD techniques are the different ways of deciding what to do when a dimension attribute changes.

---

## 3. SCD types compared

Same change in every row of this table: Asha (101) moves from Pune to Bengaluru on 2026-03-15.

| Type | Rule | What the table looks like after the change | History kept? | Use when |
| :--- | :--- | :--- | :--- | :--- |
| **0** | Never change | `101, Pune` | Original value only | Value must never change (date of birth, original signup date) |
| **1** | Overwrite | `101, Bengaluru` | None | History is not needed, or you are fixing a typo |
| **2** | Add a new row | `101, Pune, 2026-01-01 → 2026-03-15` and `101, Bengaluru, 2026-03-15 → NULL` | Full | You need "what was true at the time" — the standard choice |
| **3** | Add a column | `101, current_city=Bengaluru, previous_city=Pune` | Only one previous value | You only ever need "current vs previous" |
| **4** | Separate history table | `customers` holds current row; `customers_history` holds old rows | Full, in a second table | Current table must stay small and fast |
| **6** | Hybrid of 1 + 2 + 3 | Type 2 rows, plus a `current_city` column that is overwritten on **every** row for that customer | Full | Reports need both "as it was" and "as it is now" without an extra join |

Interviews focus on Types 1, 2, and 3. Type 2 is the one dbt snapshots implement.

In dbt terms:

- **Type 1** is an ordinary `table` model, or an `incremental` model with `unique_key` and the `merge` strategy (see [dbt-materializations-types-snowflake-guide.md](./dbt-materializations-types-snowflake-guide.md)).
- **Type 2** is a **snapshot**.

---

## 4. Anatomy of an SCD Type 2 table

```text
customer_sk | customer_id | city      | valid_from          | valid_to            | is_current
------------+-------------+-----------+---------------------+---------------------+-----------
a1f3...     | 101         | Pune      | 2026-01-01 09:00:00 | 2026-03-15 14:30:00 | FALSE
9c2e...     | 101         | Bengaluru | 2026-03-15 14:30:00 | 9999-12-31 00:00:00 | TRUE
```

| Column | Meaning |
| :--- | :--- |
| **Natural / business key** (`customer_id`) | The ID from the source system. It **repeats** — one row per version. |
| **Surrogate key** (`customer_sk`) | Unique per **version**. Fact tables store this to point at the exact version that was valid. |
| `valid_from` | When this version became true. |
| `valid_to` | When this version stopped being true. For the current version it is `NULL` or a far-future date such as `9999-12-31`. |
| `is_current` | Convenience flag for the latest version. |

**Rules that must always hold**

1. Exactly one current row per natural key (zero if the record was deleted and you close deleted records).
2. Versions of one key never overlap in time.
3. The old version's `valid_to` equals the new version's `valid_from`. The range is therefore **half-open**: `valid_from` is included, `valid_to` is excluded. Always filter with `>= valid_from AND < valid_to`, never `BETWEEN`.

---

## 5. dbt snapshots: how dbt does SCD Type 2

A **snapshot** is a dbt resource type (like a model or a seed) that records how rows of a mutable table change over time.

```text
First `dbt snapshot` run
    Creates the snapshot table = result of your query + metadata columns.
    Every row is "current" (dbt_valid_to is NULL).

Every later `dbt snapshot` run
    For each row in the source, matched to the current snapshot row by unique_key:
      - New key         -> INSERT a row
      - Changed row     -> UPDATE the old row's dbt_valid_to, then INSERT the new version
      - Unchanged row   -> do nothing
      - Key disappeared -> depends on hard_deletes (default: do nothing)
```

**Metadata columns dbt adds**

| Column | Meaning |
| :--- | :--- |
| `dbt_valid_from` | When this version became valid. |
| `dbt_valid_to` | When this version stopped being valid. `NULL` for the current version (or the value of `dbt_valid_to_current` if configured). |
| `dbt_scd_id` | A hash that is unique for each snapshot row. Use it as the version-level surrogate key. |
| `dbt_updated_at` | The change timestamp dbt recorded when it inserted this row. Used internally by dbt. |
| `dbt_is_deleted` | Only exists when `hard_deletes: new_record`. It is a **string**: `'True'` or `'False'`. |

**Key facts**

- Snapshots are run by `dbt snapshot` or `dbt build`. **`dbt run` does not run snapshots.**
- Snapshots are referenced downstream with `{{ ref('snapshot_name') }}`, exactly like a model.
- A snapshot records the state of the source **at the moment it runs**. It does not see changes that happened and were overwritten between two runs.
- Snapshots **cannot be rebuilt**. The history exists only in the snapshot table. `--full-refresh` is ignored for snapshots.

---

## 6. Full worked example

### Step 0 — Source table in Snowflake

```sql
CREATE DATABASE IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS RAW.CRM;

CREATE OR REPLACE TABLE RAW.CRM.CUSTOMERS (
    customer_id    INTEGER,
    customer_name  VARCHAR,
    city           VARCHAR,
    loyalty_tier   VARCHAR,
    updated_at     TIMESTAMP_NTZ
);

INSERT INTO RAW.CRM.CUSTOMERS VALUES
    (101, 'Asha Rao',   'Pune',   'Silver', '2026-01-01 09:00:00'),
    (102, 'Ravi Kumar', 'Delhi',  'Gold',   '2026-01-01 09:00:00'),
    (103, 'Meera Shah', 'Mumbai', 'Bronze', '2026-01-01 09:00:00');
```

### Step 1 — Declare the source

```yaml
# models/staging/crm/_crm__sources.yml
sources:
  - name: crm
    database: RAW
    schema: CRM
    tables:
      - name: customers
        columns:
          - name: customer_id
            data_tests:
              - unique
              - not_null
```

The `unique` test matters: dbt matches source rows to snapshot rows by this key. Duplicates in the source corrupt the snapshot.

### Step 2 — Define the snapshot

```yaml
# snapshots/customers_snapshot.yml
snapshots:
  - name: customers_snapshot              # name of the table that gets created
    relation: source('crm', 'customers')  # what to snapshot (a source() or a ref())
    config:
      schema: snapshots                   # see Section 10 for the exact schema name this produces
      unique_key: customer_id             # how dbt matches source rows to snapshot rows
      strategy: timestamp                 # how dbt detects a change
      updated_at: updated_at              # column used by the timestamp strategy
```

There is no `.sql` file. In dbt v1.9+ the YAML file is the whole snapshot.

### Step 3 — First run (2026-01-02 02:00)

```bash
dbt snapshot --select customers_snapshot
```

`customers_snapshot` now contains:

| customer_id | customer_name | city | loyalty_tier | updated_at | dbt_scd_id | dbt_updated_at | dbt_valid_from | dbt_valid_to |
| ---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 101 | Asha Rao | Pune | Silver | 2026-01-01 09:00 | `a1f3…` | 2026-01-01 09:00 | 2026-01-01 09:00 | NULL |
| 102 | Ravi Kumar | Delhi | Gold | 2026-01-01 09:00 | `b7d0…` | 2026-01-01 09:00 | 2026-01-01 09:00 | NULL |
| 103 | Meera Shah | Mumbai | Bronze | 2026-01-01 09:00 | `c44a…` | 2026-01-01 09:00 | 2026-01-01 09:00 | NULL |

With the `timestamp` strategy, `dbt_valid_from` is the row's own `updated_at` value. The time the job ran (02:00) is not stored anywhere.

### Step 4 — The source changes

```sql
-- Asha moves city
UPDATE RAW.CRM.CUSTOMERS
SET city = 'Bengaluru',
    updated_at = '2026-03-15 14:30:00'
WHERE customer_id = 101;

-- A new customer signs up
INSERT INTO RAW.CRM.CUSTOMERS VALUES
    (104, 'John Dsouza', 'Goa', 'Silver', '2026-03-15 16:00:00');
```

### Step 5 — Second run (2026-03-16 02:00)

```bash
dbt snapshot --select customers_snapshot
```

| customer_id | customer_name | city | loyalty_tier | updated_at | dbt_valid_from | dbt_valid_to | What happened |
| ---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 101 | Asha Rao | Pune | Silver | 2026-01-01 09:00 | 2026-01-01 09:00 | **2026-03-15 14:30** | Old version closed (UPDATE) |
| 101 | Asha Rao | **Bengaluru** | Silver | 2026-03-15 14:30 | **2026-03-15 14:30** | NULL | New version inserted |
| 102 | Ravi Kumar | Delhi | Gold | 2026-01-01 09:00 | 2026-01-01 09:00 | NULL | Unchanged, untouched |
| 103 | Meera Shah | Mumbai | Bronze | 2026-01-01 09:00 | 2026-01-01 09:00 | NULL | Unchanged, untouched |
| 104 | John Dsouza | Goa | Silver | 2026-03-15 16:00 | 2026-03-15 16:00 | NULL | New key inserted |

(`dbt_scd_id` and `dbt_updated_at` are omitted here for width; they are still in the table.)

That is SCD Type 2. The rest of this guide is configuration, querying, and operations.

---

## 7. Strategies: `timestamp` vs `check`

A strategy answers one question: **how does dbt know a row changed?**

### `timestamp` (preferred)

```yaml
config:
  unique_key: customer_id
  strategy: timestamp
  updated_at: updated_at
```

- A row is "changed" when the source `updated_at` is **later** than the `dbt_valid_from` of the current snapshot row.
- `dbt_valid_from` and `dbt_valid_to` are taken from the source `updated_at`, so they reflect when the change really happened.
- New columns in the source need no config change.
- **Requirement:** `updated_at` must be updated on every change. If a row changes but `updated_at` does not, the change is missed.

### `check`

Use it when the source has no reliable `updated_at` column.

```yaml
# snapshots/customers_snapshot_check.yml
snapshots:
  - name: customers_snapshot_check
    relation: source('crm', 'customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: check
      check_cols:
        - city
        - loyalty_tier
```

- A row is "changed" when the value of **any** column in `check_cols` differs from the current snapshot row.
- Columns not listed are ignored. With the config above, a change to `customer_name` alone creates no new version.
- `check_cols: all` compares every column. Listing columns explicitly is safer.
- `dbt_valid_from` and `dbt_valid_to` are the **time the snapshot ran**, because dbt has no other timestamp to use.

The same scenario from Section 6 under the `check` strategy:

| customer_id | city | dbt_valid_from | dbt_valid_to |
| ---: | :--- | :--- | :--- |
| 101 | Pune | 2026-01-02 02:00 | 2026-03-16 02:00 |
| 101 | Bengaluru | 2026-03-16 02:00 | NULL |

Asha really moved on 2026-03-15 14:30, but the snapshot says 2026-03-16 02:00. An order she placed at 2026-03-15 20:00 would be attributed to Pune. With `check`, your history is only as precise as your snapshot schedule.

**`check` with `updated_at`:** you can add `updated_at: <column>` to a `check` snapshot. dbt still detects changes by comparing `check_cols`, but uses that column for the validity timestamps instead of the run time (falling back to the run time when the column is `NULL`).

### Comparison

| | `timestamp` | `check` |
| :--- | :--- | :--- |
| Needs | A reliable `updated_at` column | A list of columns to compare |
| Detects a change when | `updated_at` is newer | Any listed column value differs |
| `dbt_valid_from` / `dbt_valid_to` come from | Source `updated_at` | Snapshot run time |
| Cost | Compares one column | Compares every listed column |
| Source adds a column | Nothing to do | Must add it to `check_cols` if you want it tracked |
| Choose when | Whenever possible | No trustworthy `updated_at` exists |

---

## 8. Hard deletes

A **hard delete** is a row physically removed from the source (`DELETE FROM ...`). A **soft delete** is a flag such as `is_deleted = TRUE`; that is just a normal column change and needs no special handling.

`hard_deletes` (dbt v1.9+) has three options and works with either strategy:

```yaml
config:
  unique_key: customer_id
  strategy: timestamp
  updated_at: updated_at
  hard_deletes: invalidate      # ignore (default) | invalidate | new_record
```

Scenario: Meera (103) is deleted from the source on 2026-06-01. The snapshot runs on 2026-06-02 02:00. Her rows in the snapshot afterwards:

**`ignore` (default)** — nothing happens. She looks like an active customer forever.

| customer_id | city | dbt_valid_from | dbt_valid_to |
| ---: | :--- | :--- | :--- |
| 103 | Mumbai | 2026-01-01 09:00 | NULL |

**`invalidate`** — her current row is closed. She has no current row any more.

| customer_id | city | dbt_valid_from | dbt_valid_to |
| ---: | :--- | :--- | :--- |
| 103 | Mumbai | 2026-01-01 09:00 | 2026-06-02 02:00 |

**`new_record`** — her current row is closed **and** a marker row is inserted. The marker row repeats her last known values. A `dbt_is_deleted` column is added to the whole table.

| customer_id | city | dbt_valid_from | dbt_valid_to | dbt_is_deleted |
| ---: | :--- | :--- | :--- | :--- |
| 103 | Mumbai | 2026-01-01 09:00 | 2026-06-02 02:00 | False |
| 103 | Mumbai | 2026-06-02 02:00 | NULL | True |

Points to remember:

- The delete time is always the **snapshot run time**. A deleted row has no `updated_at` to read, so dbt cannot know when the delete really happened.
- If the row comes back later, dbt inserts it as a new version. With `invalidate` there is a **gap** in that key's timeline between the delete and the return. With `new_record` the timeline is continuous.
- `dbt_is_deleted` holds the strings `'True'` / `'False'`, not booleans. Filter with `dbt_is_deleted = 'False'`.
- Deletes are detected by comparing the full set of source keys with the snapshot. Your snapshot query must therefore return **all** rows. If you filter the source (for example "last 7 days only"), every row outside the filter is treated as deleted.
- Do not change `hard_deletes` on an existing snapshot without migrating the table. dbt does not migrate it for you. `new_record` in particular needs the `dbt_is_deleted` column to exist.

---

## 9. Other snapshot configs

### `dbt_valid_to_current` — a far-future date instead of `NULL` (v1.9+)

```yaml
config:
  unique_key: customer_id
  strategy: timestamp
  updated_at: updated_at
  dbt_valid_to_current: "to_date('9999-12-31')"
```

Current rows get `dbt_valid_to = 9999-12-31` instead of `NULL`, so range joins need no `COALESCE`.

- The value is a **SQL expression written as a string**. It must be static; Jinja such as `{{ var(...) }}` is not supported.
- dbt does **not** rewrite existing rows. If you add this to an existing snapshot, old current rows keep `NULL` and new ones get the date. Either set it from the very first run, or update the existing `NULL` rows by hand.

### `snapshot_meta_column_names` — rename the metadata columns (v1.9+)

```yaml
config:
  unique_key: customer_id
  strategy: timestamp
  updated_at: updated_at
  snapshot_meta_column_names:
    dbt_valid_from: valid_from
    dbt_valid_to: valid_to
    dbt_scd_id: scd_id
    dbt_updated_at: modified_at
    dbt_is_deleted: is_deleted
```

Set this before the first run. dbt does not rename columns in an existing snapshot table, and the next run errors if the names do not match.

### Composite `unique_key`

```yaml
# dbt v1.9+: a list of columns
config:
  unique_key:
    - order_id
    - product_id
```

In dbt v1.8 and earlier, `unique_key` accepts only a single column or SQL expression, so build one: `unique_key="order_id || '-' || product_id"`.

### Snapshotting a transformed query

`relation` accepts only `source(...)` or `ref(...)`. If you must filter or deduplicate first, put that logic in an ephemeral model and snapshot it:

```sql
-- models/staging/crm/int_customers_deduped.sql
{{ config(materialized='ephemeral') }}

SELECT *
FROM {{ source('crm', 'customers') }}
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) = 1
```

```yaml
# snapshots/customers_snapshot.yml
snapshots:
  - name: customers_snapshot
    relation: ref('int_customers_deduped')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp
      updated_at: updated_at
```

Keep such logic to the minimum. Anything you transform away before the snapshot is lost from history.

### Project-level defaults

```yaml
# dbt_project.yml
snapshot-paths: ["snapshots"]

snapshots:
  my_analytics_project:
    +schema: snapshots
    +tags: ["snapshot"]
```

---

## 10. Where snapshots are built (schemas and environments)

This differs between the new and legacy syntax and is a common source of confusion.

| Config | Syntax | Resulting schema |
| :--- | :--- | :--- |
| `schema: snapshots` | v1.9+ | Passed through `generate_schema_name`, like models. With dbt's default macro that is `<target_schema>_snapshots`, for example `dbt_dev_snapshots` in dev and `analytics_snapshots` in prod. |
| *(no schema config)* | v1.9+ | Your target schema, like any model. |
| `target_schema='snapshots'` | Legacy | Exactly `snapshots` in **every** environment. Dev and prod write to the **same table** unless you add Jinja to vary it. |

The v1.9+ behaviour is described as "environment-aware": each developer gets their own snapshot table, and prod history is not touched from dev. `database` works the same way through `generate_database_name`.

If your project overrides `generate_schema_name` (many do, so that prod uses the bare custom schema name), the snapshot follows that override.

---

## 11. Legacy syntax (dbt v1.8 and earlier)

Before v1.9 a snapshot was a `.sql` file in the `snapshots/` directory, containing a `select` wrapped in a `{% snapshot %}` block. It still works in current versions.

```sql
-- snapshots/customers_snapshot.sql
{% snapshot customers_snapshot %}

{{
    config(
        target_database='ANALYTICS',
        target_schema='SNAPSHOTS',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at',
        invalidate_hard_deletes=True
    )
}}

SELECT * FROM {{ source('crm', 'customers') }}

{% endsnapshot %}
```

`check` strategy in the legacy syntax:

```sql
-- snapshots/customers_snapshot_check.sql
{% snapshot customers_snapshot_check %}

{{
    config(
        target_schema='SNAPSHOTS',
        unique_key='customer_id',
        strategy='check',
        check_cols=['city', 'loyalty_tier']
    )
}}

SELECT * FROM {{ source('crm', 'customers') }}

{% endsnapshot %}
```

| | Legacy (≤ v1.8) | Current (v1.9+) |
| :--- | :--- | :--- |
| Defined in | `.sql` file with `{% snapshot %}` block | `.yml` file with `relation:` |
| Schema | `target_schema` (**required**) | `schema` (optional, environment-aware) |
| Database | `target_database` | `database` |
| Hard deletes | `invalidate_hard_deletes=True` | `hard_deletes: ignore / invalidate / new_record` |
| Transformations | Written inside the block | A separate ephemeral model |
| `dbt_valid_to_current`, `snapshot_meta_column_names`, list `unique_key` | Not available | Available |

`invalidate_hard_deletes=True` behaves the same as `hard_deletes: invalidate`. The two cannot be set together. Define each snapshot name once — in a `.sql` file or a `.yml` file, not both.

---

## 12. Building the dimension on top of the snapshot

Keep the snapshot raw. Put renaming, the surrogate key, and the current flag in a model on top of it.

```sql
-- models/marts/dim_customers.sql
{{ config(materialized='table') }}

WITH snap AS (
    SELECT * FROM {{ ref('customers_snapshot') }}
)

SELECT
    dbt_scd_id                                            AS customer_sk,   -- unique per version
    customer_id,                                                            -- natural key, repeats
    customer_name,
    city,
    loyalty_tier,
    dbt_valid_from                                        AS valid_from,
    COALESCE(dbt_valid_to, '9999-12-31'::TIMESTAMP_NTZ)   AS valid_to,
    dbt_valid_to IS NULL                                  AS is_current
FROM snap
```

Result after the second run in Section 6:

| customer_sk | customer_id | customer_name | city | loyalty_tier | valid_from | valid_to | is_current |
| :--- | ---: | :--- | :--- | :--- | :--- | :--- | :--- |
| `a1f3…` | 101 | Asha Rao | Pune | Silver | 2026-01-01 09:00 | 2026-03-15 14:30 | FALSE |
| `9c2e…` | 101 | Asha Rao | Bengaluru | Silver | 2026-03-15 14:30 | 9999-12-31 00:00 | TRUE |
| `b7d0…` | 102 | Ravi Kumar | Delhi | Gold | 2026-01-01 09:00 | 9999-12-31 00:00 | TRUE |
| `c44a…` | 103 | Meera Shah | Mumbai | Bronze | 2026-01-01 09:00 | 9999-12-31 00:00 | TRUE |
| `d810…` | 104 | John Dsouza | Goa | Silver | 2026-03-15 16:00 | 9999-12-31 00:00 | TRUE |

Adjust two lines if your snapshot config differs:

- With `dbt_valid_to_current: "to_date('9999-12-31')"`, use `dbt_valid_to AS valid_to` and `dbt_valid_to = '9999-12-31'::TIMESTAMP_NTZ AS is_current`.
- With `hard_deletes: new_record`, add `WHERE dbt_is_deleted = 'False'` (or expose the column) so delete-marker rows do not appear as current customers.

**Optional — backdating the first version.** History starts at the first `dbt_valid_from`. A fact dated before that finds no matching version. If you accept the assumption "the first value we saw was also true earlier", replace the `valid_from` line with:

```sql
    CASE
        WHEN ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY dbt_valid_from) = 1
            THEN '1900-01-01'::TIMESTAMP_NTZ
        ELSE dbt_valid_from
    END                                                   AS valid_from,
```

---

## 13. Querying an SCD Type 2 table

Sample fact data used below:

| order_id | customer_id | order_ts | amount |
| ---: | ---: | :--- | ---: |
| 5001 | 101 | 2026-02-10 11:00 | 1000 |
| 5002 | 101 | 2026-04-20 18:00 | 2500 |

### A. Current state only

```sql
SELECT customer_id, customer_name, city, loyalty_tier
FROM {{ ref('dim_customers') }}
WHERE is_current
```

### B. State as of a past moment

```sql
SELECT customer_id, customer_name, city, loyalty_tier
FROM {{ ref('dim_customers') }}
WHERE '2026-02-01'::TIMESTAMP_NTZ >= valid_from
  AND '2026-02-01'::TIMESTAMP_NTZ <  valid_to
```

Returns Asha in Pune, and does not return John (he did not exist yet).

### C. Point-in-time join — attach the version that was valid when the fact happened

```sql
-- models/marts/fct_orders.sql
{{ config(materialized='table') }}

SELECT
    o.order_id,
    o.order_ts,
    o.amount,
    o.customer_id,
    d.customer_sk                 -- the exact customer version at order time
FROM {{ ref('stg_orders') }} AS o
LEFT JOIN {{ ref('dim_customers') }} AS d
    ON  o.customer_id = d.customer_id
    AND o.order_ts >= d.valid_from
    AND o.order_ts <  d.valid_to
```

| order_id | order_ts | amount | customer_sk | (version it points to) |
| ---: | :--- | ---: | :--- | :--- |
| 5001 | 2026-02-10 11:00 | 1000 | `a1f3…` | Pune |
| 5002 | 2026-04-20 18:00 | 2500 | `9c2e…` | Bengaluru |

Use `>=` and `<`. `BETWEEN` includes both ends, and because the old `valid_to` equals the new `valid_from`, an order at exactly `2026-03-15 14:30:00` would match both versions and be double-counted.

### D. "As it was" vs "as it is now" reporting

```sql
-- As it was: revenue by the city the customer lived in at order time
SELECT d.city, SUM(f.amount) AS revenue
FROM {{ ref('fct_orders') }} AS f
JOIN {{ ref('dim_customers') }} AS d
    ON f.customer_sk = d.customer_sk
GROUP BY d.city
-- Pune 1000, Bengaluru 2500

-- As it is now: revenue by the customer's current city
SELECT d.city, SUM(f.amount) AS revenue
FROM {{ ref('fct_orders') }} AS f
JOIN {{ ref('dim_customers') }} AS d
    ON  f.customer_id = d.customer_id
    AND d.is_current
GROUP BY d.city
-- Bengaluru 3500
```

This is why the fact keeps both `customer_sk` (a specific version) and `customer_id` (the customer regardless of version).

---

## 14. Testing

### Generic tests on the snapshot

```yaml
# snapshots/customers_snapshot.yml
snapshots:
  - name: customers_snapshot
    relation: source('crm', 'customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp
      updated_at: updated_at
    columns:
      - name: dbt_scd_id
        data_tests:
          - unique
          - not_null
      - name: customer_id
        data_tests:
          - not_null
      - name: dbt_valid_from
        data_tests:
          - not_null
```

`data_tests:` is the key name from dbt v1.8 onward. In earlier versions it is `tests:`.

### Singular test — at most one current row per key

```sql
-- tests/assert_one_current_row_per_customer.sql
-- Fails if any customer has more than one current version.
SELECT
    customer_id,
    COUNT(*) AS current_rows
FROM {{ ref('customers_snapshot') }}
WHERE dbt_valid_to IS NULL
GROUP BY customer_id
HAVING COUNT(*) > 1
```

### Singular test — versions do not overlap

```sql
-- tests/assert_no_overlapping_customer_versions.sql
-- Fails if a version ends after the next version of the same customer starts.
WITH versions AS (
    SELECT
        customer_id,
        valid_from,
        valid_to,
        LEAD(valid_from) OVER (PARTITION BY customer_id ORDER BY valid_from) AS next_valid_from
    FROM {{ ref('dim_customers') }}
)

SELECT *
FROM versions
WHERE next_valid_from IS NOT NULL
  AND valid_to > next_valid_from
```

A singular test passes when its query returns zero rows. The `dbt_utils` package also ships a generic test for this, `mutually_exclusive_ranges`.

The most valuable test is still the `unique` + `not_null` test on the **source** key (Section 6, Step 1). Run it before the snapshot so duplicates are caught before they are written into history.

---

## 15. Running and scheduling

| Command | What it does |
| :--- | :--- |
| `dbt snapshot` | Runs all snapshots |
| `dbt snapshot --select customers_snapshot` | Runs one snapshot |
| `dbt build` | Runs seeds, snapshots, models, and tests in DAG order |
| `dbt run` | Runs models only — **snapshots are not run** |
| `dbt test --select customers_snapshot` | Runs the tests defined on the snapshot |
| `dbt snapshot --full-refresh` | Runs normally; the flag is ignored and history is kept |

From dbt v1.12, `dbt compile --select customers_snapshot` writes the generated snapshot SQL to `target/compiled/`.

**How often?** As often as you need to see changes. A snapshot only records what the source looks like when it runs:

```text
Run at 02:00  -> sees Pune
  10:00  Pune -> Hyderabad      (never seen)
  17:00  Hyderabad -> Bengaluru
Run at 02:00  -> sees Bengaluru

Snapshot history: Pune -> Bengaluru.   Hyderabad was never recorded.
```

If every intermediate change matters, snapshots are the wrong tool; you need a change log from the source (CDC). See Section 16.

**Typical production order**

```bash
dbt source freshness                 # optional: is the source loaded?
dbt test --select source:crm         # source key is unique and not null
dbt snapshot                         # capture history
dbt run                              # build models on top of the snapshots
dbt test                             # test models
```

Many teams run `dbt snapshot` as its own job, so that a failing model elsewhere never stops history from being captured.

**To start a snapshot over,** you must drop the table yourself (`DROP TABLE ...`) and run `dbt snapshot` again. All history collected so far is lost.

---

## 16. SCD Type 2 without snapshots

### A. What a snapshot does, written by hand in Snowflake

Useful for understanding the mechanics and for the interview question "how would you do it without dbt snapshots?".

```sql
CREATE TABLE IF NOT EXISTS ANALYTICS.CORE.CUSTOMERS_SCD2 (
    customer_id    INTEGER,
    customer_name  VARCHAR,
    city           VARCHAR,
    loyalty_tier   VARCHAR,
    updated_at     TIMESTAMP_NTZ,
    valid_from     TIMESTAMP_NTZ,
    valid_to       TIMESTAMP_NTZ,
    is_current     BOOLEAN
);

USE SCHEMA ANALYTICS.CORE;
```

**Two statements (easiest to read)**

```sql
BEGIN;

-- Step 1: close the current version of every customer whose source row is newer
UPDATE CUSTOMERS_SCD2
SET valid_to   = src.updated_at,
    is_current = FALSE
FROM RAW.CRM.CUSTOMERS AS src
WHERE CUSTOMERS_SCD2.customer_id = src.customer_id
  AND CUSTOMERS_SCD2.is_current  = TRUE
  AND src.updated_at > CUSTOMERS_SCD2.valid_from;

-- Step 2: insert a current version for every customer that has none.
-- That covers brand-new customers AND the ones just closed in Step 1.
INSERT INTO CUSTOMERS_SCD2
    (customer_id, customer_name, city, loyalty_tier, updated_at, valid_from, valid_to, is_current)
SELECT
    src.customer_id,
    src.customer_name,
    src.city,
    src.loyalty_tier,
    src.updated_at,
    src.updated_at,
    NULL,
    TRUE
FROM RAW.CRM.CUSTOMERS AS src
LEFT JOIN CUSTOMERS_SCD2 AS cur
    ON  cur.customer_id = src.customer_id
    AND cur.is_current  = TRUE
WHERE cur.customer_id IS NULL;

COMMIT;
```

**One `MERGE` (the classic interview version)**

A single `MERGE` can either update or insert for one source row, but a changed customer needs both. The trick is to feed changed rows in **twice**: once with the real key (to close the old version) and once with a `NULL` key (which can never match, so it is inserted).

```sql
MERGE INTO CUSTOMERS_SCD2 AS tgt
USING (
    -- (a) every source row with its real key:
    --     matches the current version if the customer exists, otherwise inserts a new customer
    SELECT src.customer_id AS merge_key, src.*
    FROM RAW.CRM.CUSTOMERS AS src

    UNION ALL

    -- (b) changed rows again with a NULL key: never matches, so always inserted as the new version
    SELECT NULL AS merge_key, src.*
    FROM RAW.CRM.CUSTOMERS AS src
    JOIN CUSTOMERS_SCD2 AS cur
        ON  cur.customer_id = src.customer_id
        AND cur.is_current  = TRUE
    WHERE src.updated_at > cur.valid_from
) AS s
    ON  tgt.customer_id = s.merge_key
    AND tgt.is_current  = TRUE

WHEN MATCHED AND s.updated_at > tgt.valid_from THEN
    UPDATE SET
        valid_to   = s.updated_at,
        is_current = FALSE

WHEN NOT MATCHED THEN
    INSERT (customer_id, customer_name, city, loyalty_tier, updated_at, valid_from, valid_to, is_current)
    VALUES (s.customer_id, s.customer_name, s.city, s.loyalty_tier, s.updated_at, s.updated_at, NULL, TRUE);
```

dbt's snapshot does the same job: it builds a temporary staging table of rows labelled insert / update / delete, then applies it to the snapshot table with one `MERGE`.

### B. When the source already gives you a change log (CDC)

If the source delivers **one row per change** (Fivetran history mode, Debezium, a Snowflake stream landed into a table, an audit table), the history already exists. You do not need a snapshot. Derive the validity range with a window function in a normal model:

| customer_id | city | changed_at |
| ---: | :--- | :--- |
| 101 | Pune | 2026-01-01 09:00 |
| 101 | Hyderabad | 2026-03-15 10:00 |
| 101 | Bengaluru | 2026-03-15 17:00 |

```sql
-- models/marts/dim_customers_from_cdc.sql
{{ config(materialized='table') }}

WITH changes AS (
    SELECT * FROM {{ ref('stg_crm__customer_changes') }}   -- one row per change
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['customer_id', 'changed_at']) }}  AS customer_sk,
    customer_id,
    customer_name,
    city,
    loyalty_tier,
    changed_at                                                              AS valid_from,
    COALESCE(
        LEAD(changed_at) OVER (PARTITION BY customer_id ORDER BY changed_at),
        '9999-12-31'::TIMESTAMP_NTZ
    )                                                                       AS valid_to,
    LEAD(changed_at) OVER (PARTITION BY customer_id ORDER BY changed_at) IS NULL
                                                                            AS is_current
FROM changes
```

`generate_surrogate_key` comes from the `dbt_utils` package:

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

```bash
dbt deps
```

### Snapshot vs CDC-based model

| | Snapshot | Model over a change log |
| :--- | :--- | :--- |
| Source provides | Current state only | Every change |
| Captures changes between runs | No, only the latest state | Yes, all of them |
| Rebuildable from source | **No** | Yes — `dbt run --full-refresh` is safe |
| Validity timestamps | `updated_at` or run time | Real change time |
| Use when | Source overwrites and keeps no history | A change log exists |

Rule of thumb: **if the source keeps history, model it; if the source overwrites, snapshot it.**

---

## 17. Gotchas

1. **Changes between runs are lost.** Only the state at run time is recorded (Section 15).
2. **Snapshots cannot be rebuilt.** Dropping the table loses history permanently. `--full-refresh` does not rebuild it.
3. **`dbt run` skips snapshots.** Use `dbt snapshot` or `dbt build`.
4. **Duplicate `unique_key` values in the source** produce duplicate versions or a failing `MERGE`. Test the source key.
5. **`timestamp` strategy trusts `updated_at`.** A change that does not bump `updated_at` is invisible. A backfill that bumps `updated_at` on every row creates a new version of every row.
6. **`check` strategy timestamps are run times,** not real change times (Section 7).
7. **`check_cols` only watches listed columns.** With `check_cols: all`, adding a column to the source makes dbt treat every row as changed on the next run, creating one new version per row.
8. **Default `hard_deletes` is `ignore`.** Deleted source rows stay "current" unless you configure otherwise.
9. **Filtering the snapshot query + hard-delete tracking** closes every row outside the filter (Section 8).
10. **Schema changes in the source:**
    - New column → dbt adds it to the snapshot; older versions hold `NULL`.
    - Removed column → dbt keeps it in the snapshot; newer versions hold `NULL`.
    - Changed data type → dbt does not alter the column type (it only widens string columns where the platform needs it).
11. **Adding `dbt_valid_to_current`, `snapshot_meta_column_names`, or `hard_deletes` to an existing snapshot** needs a manual table migration. dbt does not do it.
12. **Legacy `target_schema`** points dev and prod at the same table (Section 10).
13. **`BETWEEN` in a range join** double-matches on the boundary. Use `>=` and `<`.
14. **Mixed time zones.** The fact timestamp and the snapshot validity columns must be the same type and time zone, or point-in-time joins shift.
15. **History starts at the first snapshot.** You cannot recover what the source looked like before you began snapshotting.

---

## 18. Best practices

1. **Snapshot the raw source with `select *`.** Do cleaning and renaming downstream. You cannot recover a column you dropped before the snapshot.
2. **Prefer `timestamp`** whenever a reliable `updated_at` exists.
3. **Test the source key** for `unique` and `not_null` before the snapshot runs.
4. **Keep snapshots in a dedicated schema** with restricted privileges, so they are hard to drop by accident.
5. **Decide on hard deletes deliberately.** Do not leave the default without checking whether the source deletes rows.
6. **Run snapshots on a schedule that matches how fast the source changes,** and consider a separate job for them.
7. **Build a `dim_` model on top** with a surrogate key, `valid_from`, `valid_to`, and `is_current`. Do not let BI tools query the snapshot directly.
8. **Set `dbt_valid_to_current` and `snapshot_meta_column_names` from the first run** if you want them.
9. **Protect the table in Snowflake.** dbt-snowflake creates model tables as `TRANSIENT` by default, which have no Fail-safe. Check what your snapshot table is with `SHOW TABLES LIKE 'CUSTOMERS_SNAPSHOT' IN SCHEMA <db>.<schema>;` (the `kind` column), and keep a periodic backup, for example a zero-copy clone: `CREATE TABLE <backup_name> CLONE <snapshot_table>;`.

---

## 19. Interview questions and answers

**Q1. What is SCD Type 2?**
A technique for keeping the full history of a dimension. Every change inserts a new row and closes the previous one with validity dates, so you can tell what was true at any point in time.

**Q2. How do you implement SCD Type 2 in dbt?**
With a snapshot. Define it with a `unique_key` and a `strategy`, run it with `dbt snapshot`, and build a dimension model on top with `ref()`. See Section 1.

**Q3. What are the two snapshot strategies, and which do you prefer?**
`timestamp` compares an `updated_at` column. `check` compares a list of columns. Prefer `timestamp`: it compares one column, keeps the real change time, and needs no maintenance when columns are added. Use `check` only when no reliable `updated_at` exists.

**Q4. What columns does dbt add to a snapshot?**
`dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, `dbt_updated_at`, and `dbt_is_deleted` when `hard_deletes: new_record` is set.

**Q5. How do you get only the current records?**
`WHERE dbt_valid_to IS NULL`, or `WHERE dbt_valid_to = <your far-future value>` if `dbt_valid_to_current` is configured.

**Q6. How does dbt handle rows deleted from the source?**
By default it ignores them. `hard_deletes: invalidate` closes the current row. `hard_deletes: new_record` closes it and adds a marker row with `dbt_is_deleted = 'True'`. Before v1.9 the only option was `invalidate_hard_deletes=True`.

**Q7. What happens when you run a snapshot with `--full-refresh`?**
Nothing different. Snapshots ignore the flag so that history is never dropped.

**Q8. A record changed three times between two snapshot runs. How many versions are stored?**
One new version, holding the latest state. The two intermediate states are lost. If they matter, use CDC from the source.

**Q9. Snapshot or incremental model?**
A snapshot keeps history of a source that overwrites itself, and it cannot be rebuilt. An incremental model efficiently loads new or changed rows into a table and can be rebuilt from its source. If the source already provides every change, build SCD Type 2 with a normal or incremental model and `LEAD()`. If it only provides current state, use a snapshot.

**Q10. Why does an SCD Type 2 dimension need a surrogate key?**
The natural key repeats, once per version, so it cannot identify a single row. The surrogate key identifies one version. In dbt, `dbt_scd_id` already serves this purpose.

**Q11. How do you join a fact table to an SCD Type 2 dimension?**
On the natural key plus a half-open date range: `fact_ts >= valid_from AND fact_ts < valid_to`.

**Q12. Where should snapshots sit in the project?**
Directly on raw sources, before staging. Staging and dimension models then select from the snapshot.

**Q13. What changed in dbt v1.9?**
Snapshots are defined in YAML with `relation:`. `target_schema` is no longer required and snapshots became environment-aware. New configs were added: `hard_deletes`, `dbt_valid_to_current`, `snapshot_meta_column_names`, and a list for `unique_key`.

**Q14. Why not use Snowflake Time Travel instead?**
Time Travel is for recovery. Retention is at most 90 days (1 day on Standard edition), it gives you a table as of one moment rather than a queryable history with validity ranges, and it only covers tables inside Snowflake, not the upstream source system.

**Q15. How would you implement SCD Type 2 natively in Snowflake?**
A stream on the source table captures changes, and a task runs a `MERGE` like the one in Section 16A on a schedule. dbt snapshots give the same result with less code, plus lineage, testing, and documentation.

**Q16. How would you build Type 6 (hybrid) from a snapshot?**
Keep the Type 2 rows and add a "current value" column in the dimension model, for example `FIRST_VALUE(city) OVER (PARTITION BY customer_id ORDER BY dbt_valid_from DESC) AS current_city`.

**Q17. The source added a new column. What happens to the snapshot?**
dbt adds the column to the snapshot table. Existing versions hold `NULL` for it. With `timestamp` nothing else is needed. With `check`, add it to `check_cols` if changes to it should create versions.

**Q18. How do you test a snapshot?**
`unique` and `not_null` on `dbt_scd_id`, `unique` and `not_null` on the source key, a singular test for at most one current row per key, and a singular test for non-overlapping ranges.

---

## 20. Cheat sheet

```yaml
# snapshots/customers_snapshot.yml   (dbt v1.9+)
snapshots:
  - name: customers_snapshot
    relation: source('crm', 'customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp                              # or: check
      updated_at: updated_at                           # timestamp strategy
      # check_cols: [city, loyalty_tier]               # check strategy
      hard_deletes: invalidate                         # ignore | invalidate | new_record
      dbt_valid_to_current: "to_date('9999-12-31')"    # optional
```

```bash
dbt snapshot                               # run all snapshots
dbt snapshot --select customers_snapshot   # run one
dbt build                                  # snapshots + models + tests
```

```sql
-- Current rows
SELECT * FROM {{ ref('customers_snapshot') }} WHERE dbt_valid_to IS NULL

-- Point-in-time join
LEFT JOIN {{ ref('dim_customers') }} AS d
    ON  f.customer_id = d.customer_id
    AND f.event_ts >= d.valid_from
    AND f.event_ts <  d.valid_to
```

| Remember | |
| :--- | :--- |
| SCD Type 2 in dbt | = snapshot |
| Change detected | closes old row (`dbt_valid_to`), inserts new row |
| `timestamp` | validity = source `updated_at` — preferred |
| `check` | validity = snapshot run time |
| Deletes | `hard_deletes`, default `ignore` |
| Rebuild | not possible; `--full-refresh` is ignored |
| `dbt run` | does not run snapshots |
| Frequency | changes between runs are lost |
| Range filter | `>= valid_from AND < valid_to` |
