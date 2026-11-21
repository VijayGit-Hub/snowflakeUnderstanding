# Snowflake Time Travel, Zero-Copy Cloning & Fail-safe: Quick Revision Guide

## 1. Overview of Snowflake Time Travel
**Time Travel** allows you to access, clone, and restore historical data (changed or deleted) at any point within a defined retention window.

* **Key Clauses:**
  * `AT`: Targets the exact specified timestamp, offset, or statement boundary.
  * `BEFORE`: Targets the state of the object immediately *prior* to a specific statement (Query ID) or timestamp.
* **Parameters Supported:** `TIMESTAMP`, `OFFSET` (seconds in the past), and `STATEMENT` (Query ID).
* **Retention Period (`DATA_RETENTION_TIME_IN_DAYS`):**
  * **Default:** 1 day for all objects on all editions (set to `0` to disable Time Travel).
  * **Standard Edition:** 0 or 1 day for all object types.
  * **Enterprise Edition+:** Up to **90 days** for permanent tables, schemas, and databases (you must explicitly raise it from the default of 1).
  * **Temporary & transient objects:** Always 0 or 1 day only, **on every edition** (including Enterprise). Temporary tables are also purged when the session ends.

  | Object type | Standard Edition | Enterprise Edition+ | Fail-safe |
  |---|---|---|---|
  | Permanent | 0–1 day | 0–90 days | 7 days |
  | Transient | 0–1 day | 0–1 day | 0 days |
  | Temporary | 0–1 day | 0–1 day | 0 days |

### SQL Examples: Retention Period
```sql
-- Permanent table: default retention is 1 day
CREATE TABLE orders (id INT, amount NUMBER);
SHOW TABLES LIKE 'orders';                                -- retention_time = 1

-- Permanent table on Enterprise+: can be raised up to 90 days
ALTER TABLE orders SET DATA_RETENTION_TIME_IN_DAYS = 90;  -- OK on Enterprise+
                                                          -- Error on Standard (max is 1)

-- Transient table: only 0 or 1 is allowed, even on Enterprise+
CREATE TRANSIENT TABLE stg_orders (id INT) DATA_RETENTION_TIME_IN_DAYS = 1;  -- OK
ALTER TABLE stg_orders SET DATA_RETENTION_TIME_IN_DAYS = 0;                  -- OK (Time Travel disabled)
ALTER TABLE stg_orders SET DATA_RETENTION_TIME_IN_DAYS = 7;                  -- Error: max is 1 for transient

-- Temporary table: only 0 or 1, and it disappears when the session ends
CREATE TEMPORARY TABLE tmp_orders (id INT) DATA_RETENTION_TIME_IN_DAYS = 1;  -- OK
CREATE TEMPORARY TABLE tmp_bad (id INT) DATA_RETENTION_TIME_IN_DAYS = 30;    -- Error: max is 1 for temporary

-- Retention = 0 means no Time Travel and no UNDROP
ALTER TABLE stg_orders SET DATA_RETENTION_TIME_IN_DAYS = 0;
SELECT * FROM stg_orders AT (OFFSET => -60);   -- Error: data not available
DROP TABLE stg_orders;
UNDROP TABLE stg_orders;                       -- Error: cannot be recovered

-- Check the retention setting for a table
SHOW TABLES LIKE 'stg_orders';                 -- see the retention_time column
```

* **Object Recovery (`UNDROP`):** Restores dropped tables, schemas, or databases before the Time Travel retention period expires.

### SQL Examples: UNDROP
```sql
-- 1. Basic: drop and restore a table (within retention period)
CREATE TABLE customers (id INT, name STRING);
INSERT INTO customers VALUES (1, 'Asha'), (2, 'Ravi');
DROP TABLE customers;
UNDROP TABLE customers;                      -- Table restored with its data
SELECT * FROM customers;                     -- Returns 2 rows

-- 2. See dropped tables (dropped_on column is NOT NULL for dropped ones)
SHOW TABLES HISTORY LIKE 'customers';

-- 3. Name conflict: a new table with the same name already exists
DROP TABLE customers;
CREATE TABLE customers (id INT, name STRING);  -- new empty table, same name
UNDROP TABLE customers;                        -- Error: object already exists
ALTER TABLE customers RENAME TO customers_new; -- move the new one out of the way
UNDROP TABLE customers;                        -- OK: old table restored

-- 4. Multiple drops of the same name: UNDROP restores the most recently dropped version first
DROP TABLE customers;                          -- version A dropped
CREATE TABLE customers (id INT);               -- version B
DROP TABLE customers;                          -- version B dropped
UNDROP TABLE customers;                        -- restores version B
ALTER TABLE customers RENAME TO customers_b;
UNDROP TABLE customers;                        -- restores version A

-- 5. Schema and database
DROP SCHEMA sales;
UNDROP SCHEMA sales;                           -- restores schema and all its tables
DROP DATABASE analytics_db;
UNDROP DATABASE analytics_db;                  -- restores database, schemas and tables

-- 6. Transient table with retention = 1 (works within 1 day)
CREATE TRANSIENT TABLE stg_customers (id INT) DATA_RETENTION_TIME_IN_DAYS = 1;
DROP TABLE stg_customers;
UNDROP TABLE stg_customers;                    -- OK if run within 1 day
                                               -- After 1 day: Error, and no Fail-safe for transient

-- 7. Retention = 0: UNDROP is not possible
CREATE TRANSIENT TABLE tmp_load (id INT) DATA_RETENTION_TIME_IN_DAYS = 0;
DROP TABLE tmp_load;
UNDROP TABLE tmp_load;                         -- Error: table is gone immediately

-- 8. Permanent table after retention expires
--    UNDROP fails; data is in Fail-safe (7 days) and only Snowflake Support can recover it
```

---

## 2. Point-in-Time Zero-Copy Cloning
Snowflake stores table data in immutable micro-partitions. When you combine **Zero-Copy Cloning** with **Time Travel**, you create an instant, writable copy of an object as it existed in the past.

* **Metadata-Only Operation:** Snowflake does not duplicate underlying data files; the clone points to the exact historical micro-partitions.
* **Zero Initial Storage Cost:** You pay no extra storage at creation since the original and clone share micro-partitions.
* **Independent Lifecycle:** Changes made to the clone create new micro-partitions only for the clone, leaving the original object unaffected.

### SQL Examples: Point-in-Time Cloning
```sql
-- Clone a table as it existed at a specific timestamp
CREATE TABLE orders_restored CLONE orders 
  AT (TIMESTAMP => '2026-11-21 10:00:00'::timestamp);

-- Clone a table right before an accidental DELETE or UPDATE statement ran
CREATE TABLE orders_restored CLONE orders 
  BEFORE (STATEMENT => '01b8a123-0000-4567-0000-00001234abcd');
```

---

## 3. Fail-safe & The Data Lifecycle
**Fail-safe** is a non-configurable **7-day** emergency recovery window that begins *after* Time Travel expires (applicable to permanent tables only; transient and temporary tables have 0 Fail-safe days).

* **Not Self-Service:** Once data enters Fail-safe, Time Travel queries (`AT`/`BEFORE`), `CLONE`, and `UNDROP` will **no longer work**.
* **Snowflake Support Only:** Fail-safe is strictly for catastrophic data recovery by Snowflake Support and can take hours or days.

### Lifecycle Timeline Example (Standard 1-Day Retention)
Suppose rows are accidentally deleted from a permanent table on **Nov 1 at 9:00 AM**:

| Phase | Duration | Date & Time Window | Time Travel Works? | Who Can Recover Data? |
| :--- | :--- | :--- | :--- | :--- |
| **1. Time Travel** | 1 Day | Nov 1 (9:00 AM) – Nov 2 (9:00 AM) | **Yes** | **You** (Self-service SQL: `AT`/`BEFORE`, `CLONE`, `UNDROP`) |
| **2. Fail-safe** | 7 Days (Fixed) | Nov 2 (9:00 AM) – Nov 9 (9:00 AM) | **No** | **Snowflake Support only** (Emergency recovery) |
| **3. Purged** | Permanent | After Nov 9 (9:00 AM) | **No** | **Nobody** (Data is permanently erased) |

*(Note: On Enterprise Edition with a 10-day Time Travel setting, Fail-safe still adds 7 days afterward, resulting in a 17-day total lifecycle.)*

---

## 4. Restoring Data Back into Original Tables

### Option A: Selective Recovery Using `MERGE` (Safest for Live Production Tables)
Use `MERGE` when new transactions have arrived *after* the mistake and you want to restore deleted or overwritten rows without losing recent data.

```sql
-- Step 1: Create a point-in-time clone before the accidental change
CREATE OR REPLACE TABLE orders_backup CLONE orders 
  BEFORE (STATEMENT => '01b8a123-0000-4567-0000-00001234abcd');

-- Step 2: Merge recovered rows back into the live table
MERGE INTO orders AS target
USING orders_backup AS source
  ON target.order_id = source.order_id

-- Revert rows that were accidentally overwritten
WHEN MATCHED AND target.status != source.status THEN
  UPDATE SET 
    target.status = source.status,
    target.amount = source.amount

-- Re-insert rows that were accidentally deleted
WHEN NOT MATCHED THEN
  INSERT (order_id, customer_id, status, amount, order_date)
  VALUES (source.order_id, source.customer_id, source.status, source.amount, source.order_date);

-- Step 3: Clean up the temporary backup clone
DROP TABLE orders_backup;
```

### Option B: Full Table Rollback Using `SWAP WITH`
Use `SWAP WITH` when the entire table is corrupted and no valid updates occurred after the incident. It swaps metadata instantaneously without copying data.

```sql
-- Step 1: Create a clone from the desired point in time
CREATE OR REPLACE TABLE orders_restored CLONE orders 
  AT (TIMESTAMP => '2026-11-21 10:00:00'::timestamp);

-- Step 2: Swap the restored clone with the live table
ALTER TABLE orders SWAP WITH orders_restored;

-- Step 3: Drop the old corrupted table (now renamed orders_restored)
DROP TABLE orders_restored;
```

> **Important Caveat for `SWAP WITH`:** Table-level `GRANT` privileges swap along with the table. Always verify permissions on the live table after swapping if the clone was created under a different role.
