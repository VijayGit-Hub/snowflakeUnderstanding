# Snowflake SQL `UNION` & `UNION ALL` Guide

In Snowflake SQL, **`UNION` stacks the rows from two or more tables vertically into a single result set**, rather than combining them side-by-side like a `JOIN`. Think of it like taking two spreadsheets with the same columns and pasting the rows from the second spreadsheet directly underneath the first one.

---

## 1. `UNION` vs. `UNION ALL`

Snowflake offers two primary ways to stack data, and choosing the right one affects both your output and compute costs:

| Feature | `UNION` (or `UNION DISTINCT`) | `UNION ALL` |
| :--- | :--- | :--- |
| **Duplicate rows** | **Removes** duplicate rows across the combined result. | **Keeps** all rows, including exact duplicates. |
| **Performance** | Slower (Snowflake must sort and scan all data to find duplicates). | **Faster** (Snowflake simply stacks the data without checking it). |
| **Best used when** | You specifically need a clean list of unique records. | You know there are no duplicates, or you want to preserve every row. |

> **Best Practice:** In Snowflake, default to **`UNION ALL`** unless you specifically need duplicates removed, as it avoids expensive deduplication scans and saves compute credits.

---

## 2. Basic Example

Imagine two tables tracking customer signups from different regions:

**`INDIA_CUSTOMERS`**

| ID | NAME |
| :--- | :--- |
| 1 | Aarav |
| 2 | Priya |

**`US_CUSTOMERS`**

| ID | NAME |
| :--- | :--- |
| 2 | Priya |
| 3 | John |

### Query
```sql
SELECT id, name FROM india_customers
UNION
SELECT id, name FROM us_customers;
```

### Result (using `UNION`)

| ID | NAME |
| :--- | :--- |
| 1 | Aarav |
| 2 | Priya |
| 3 | John |

*(If you used **`UNION ALL`**, Priya would appear twice in the final output.)*

---

## 3. Core Rules for Using `UNION` in Snowflake

1. **Same number of columns:** Every `SELECT` statement in the `UNION` must return the exact same number of columns.
2. **Compatible data types:** Columns must match in order and compatible data type family.
3. **Column names come from the top:** Snowflake uses the column names/aliases from your **first** `SELECT` statement for the final output header.

---

## 4. Handling Mismatched Data Types

When stacking columns with different data types, Snowflake either implicitly converts (casts) them or throws an error depending on type compatibility.

### When Snowflake Automatically Converts (Succeeds)
Snowflake follows a data type precedence hierarchy and promotes the "narrower" type to the "wider" type so no data is lost:

* **Different numeric types (`INT` + `FLOAT` / `NUMBER`):** Converts integers to match the decimal/float type.
* **Different string lengths (`VARCHAR(10)` + `VARCHAR(50)`):** Widens the output column to `VARCHAR(50)`.
* **Date and Timestamp (`DATE` + `TIMESTAMP`):** Converts `DATE` into a `TIMESTAMP` (adding `00:00:00` as the time).
* **`NULL` + Any data type:** A hardcoded `NULL` automatically adopts the data type of the matching column in the other query.

### When Stacking Fails (Throws an Error)
Snowflake will **not** implicitly convert between different data type families:

* **Text + Number (`VARCHAR` + `NUMBER`):** Fails even if the text column only contains numeric strings like `'123'`.
* **Text + Date (`VARCHAR` + `DATE`):** Cannot stack a date string like `'2026-11-22'` directly with a `DATE` column.
* **Boolean + Number (`BOOLEAN` + `INT`):** Booleans are treated strictly and do not auto-convert to `1`/`0`.
* **Semi-structured + Standard (`VARIANT` / `ARRAY` + `VARCHAR`):** Cannot be stacked without explicit conversion.

### Explicit Casting Example (Safest Approach)
```sql
SELECT 
    customer_id::VARCHAR AS id, 
    signup_date::TIMESTAMP AS created_at
FROM table_a

UNION ALL

SELECT 
    customer_id::VARCHAR AS id, 
    signup_timestamp AS created_at
FROM table_b;
```

---

## 5. Handling Missing Columns

If the second table is missing a column, a standard `UNION` will fail immediately (`invalid number of result columns for set operator input branches`). You can solve this in three ways:

### Option 1: Pad with `NULL` (Standard SQL)
Manually add `NULL` with an alias at the exact position of the missing column.

```sql
SELECT 
    id, 
    name, 
    email          -- Exists in table_a
FROM table_a

UNION ALL

SELECT 
    id, 
    name, 
    NULL AS email  -- Missing in table_b, padded with NULL
FROM table_b;
```

### Option 2: `UNION ALL BY NAME` (Snowflake Shortcut)
Snowflake supports **`UNION BY NAME`** and **`UNION ALL BY NAME`**, which match columns by **name** rather than position and automatically fill any missing columns with `NULL`.

```sql
SELECT id, name, email FROM table_a
UNION ALL BY NAME
SELECT id, name FROM table_b;
```
*(All rows from `table_b` automatically receive `NULL` for the `email` column, even if columns are listed in a different order.)*

### Option 3: `dbt_utils.union_relations` (For dbt Projects)
If you are using **dbt**, the `dbt_utils.union_relations` macro automatically handles missing columns (filling them with `NULL`) and aligns mismatched data types across multiple models/sources without manual boilerplate.

```sql
{{ dbt_utils.union_relations(
    relations=[ref('stg_india_customers'), ref('stg_us_customers')]
) }}
```
