# Snowflake Data Sharing: Quick Reference Guide

---

## 1. The Core Mental Model (No Jargon)
Think of a Snowflake **Share** like a **Google Drive folder link set to "View Only"** or a **locked glass box** placed in your front yard.

* **Your Database = Your House:** Where all your raw tables live.
* **The Share = The Locked Glass Box:** A named, metadata-only permission slip pointing to specific objects in **one primary database**.
* **The Consumer = Your Neighbor:** They walk up, look through the glass to query live data, and bring their own magnifying glass (**compute warehouse**) to read it.

### Key Rules & Boundaries
* **Zero Data Copying:** Data never leaves your account. Consumers pay **$0 storage fees** for shared data.
* **Strictly Read-Only:** Consumers cannot modify, delete, or corrupt your underlying data.
* **Live Updates:** Any changes you make to shared tables or views appear **instantaneously** for consumers.
* **One Database per Share:** A single Share can only directly expose objects from **one container database** (though there is **no hard limit** on the number of tables/views inside that database).
* **No `FUTURE` Grants:** You must explicitly grant every new or recreated object to the Share.

---

## 2. Who Are You Sharing With?
You attach consumer accounts to your Share using `ALTER SHARE ... ADD ACCOUNTS = <account_id>;`.

1. **External Clients / Customers:** Selling or delivering live data feeds without SFTP/CSV exports.
2. **Business Partners:** Sharing live inventory, logistics, or campaign metrics.
3. **Internal Departments:** Sharing data across separate Snowflake accounts (e.g., HR, Finance, Marketing).
4. **Reader Accounts (Non-Snowflake Clients):** A managed "viewing terminal" you create for clients who do not have Snowflake. *(Note: Same attachment command, but **you pay** for their compute).*

---

## 3. Secure View vs. Combined Table (Which Should You Share?)

When you want to combine multiple messy tables into **one clean dataset** for a consumer, choose between these two approaches:

| Feature | Option A: Share a **Secure View** | Option B: Share a **Combined Table** |
| :--- | :--- | :--- |
| **How it works** | A live `JOIN` query wrapped in a `SECURE VIEW`. | A scheduled task/pipeline saves pre-joined data into a single physical table. |
| **Data Freshness** | **Instant (Real-time)** | **Delayed** (Depends on refresh schedule) |
| **Query Speed** | **Slower** (Joins execute on the fly) | **Very Fast** (Reads a flat, pre-joined table) |
| **Your Storage Cost** | **$0 extra** | **Slight increase** (Stores the combined table) |
| **Best Use Case** | Small/medium joins, real-time needs, or hiding sensitive columns/SQL logic. | Heavy, complex joins across massive tables where speed matters most. |

> **Why `SECURE` View?** Snowflake **blocks standard views** from being shared. A `SECURE VIEW` hides your underlying SQL definition and prevents consumers from peeking at hidden columns or base tables.

---

## 4. Copy-Paste SQL Cheat Sheet

### A. Share a Secure View (Joins + Hides Sensitive Data)
```sql
-- 1. Create the Secure View (hiding sensitive columns like SSN)
CREATE OR REPLACE SECURE VIEW SALES_DB.PUBLIC.CUSTOMER_ORDERS_VW AS
SELECT c.cust_id, c.name, o.order_id, o.amount, o.order_date
FROM SALES_DB.PUBLIC.CUSTOMERS c
JOIN SALES_DB.PUBLIC.ORDERS o ON c.cust_id = o.cust_id;

-- 2. Create the Share and grant hierarchy permissions
CREATE SHARE CLIENT_VIEW_SHARE;
GRANT USAGE ON DATABASE SALES_DB TO SHARE CLIENT_VIEW_SHARE;
GRANT USAGE ON SCHEMA SALES_DB.PUBLIC TO SHARE CLIENT_VIEW_SHARE;

-- 3. Add ONLY the Secure View (base tables remain completely hidden)
GRANT SELECT ON VIEW SALES_DB.PUBLIC.CUSTOMER_ORDERS_VW TO SHARE CLIENT_VIEW_SHARE;

-- 4. Attach Consumer Account (Regular or Reader Account)
ALTER SHARE CLIENT_VIEW_SHARE ADD ACCOUNTS = xy12345;
```

### B. Share a Simple Table Directly
```sql
CREATE SHARE SIMPLE_TABLE_SHARE;
GRANT USAGE ON DATABASE SALES_DB TO SHARE SIMPLE_TABLE_SHARE;
GRANT USAGE ON SCHEMA SALES_DB.PUBLIC TO SHARE SIMPLE_TABLE_SHARE;
GRANT SELECT ON TABLE SALES_DB.PUBLIC.ORDERS TO SHARE SIMPLE_TABLE_SHARE;

ALTER SHARE SIMPLE_TABLE_SHARE ADD ACCOUNTS = xy12345;
```

### C. Consumer Side: How They Use Your Share
Consumers treat your shared view or table just like a normal table—they can query it, join it with their own data, or build dashboards on top of it:
```sql
-- 1. Create a local database pointing to your Share
CREATE DATABASE VENDOR_DATA FROM SHARE <provider_account>.CLIENT_VIEW_SHARE;

-- 2. Query the shared object
SELECT * FROM VENDOR_DATA.PUBLIC.CUSTOMER_ORDERS_VW WHERE amount > 500;
```

### D. Create & Attach a Reader Account (Quick Preview)
```sql
-- 1. Spin up a managed Reader Account for a non-Snowflake client
CREATE MANAGED ACCOUNT client_reader_acct
    ADMIN_NAME = client_admin,
    ADMIN_PASSWORD = 'StrongPassword123!',
    TYPE = READER;

-- 2. Attach it just like any normal account
ALTER SHARE CLIENT_VIEW_SHARE ADD ACCOUNTS = client_reader_acct;
```

---

## 5. Key Caveats & Rules (Copy-Paste One-Liners)

| Rule | What It Means |
|------|---------------|
| **One-Database Boundary** | A single Share can only expose **one primary container database**. To combine data from multiple databases, create a **Secure View** in your primary shared database and grant `REFERENCE_USAGE` on the secondary databases to the share. |
| **Why `SECURE` Views Are Mandatory** | Snowflake blocks standard views from shares because a consumer could inspect the underlying SQL definition or use query-optimization tricks to infer hidden rows/columns. The `SECURE` keyword hides your code and locks down data privacy. |
| **No `FUTURE` Grants Allowed** | You cannot use `GRANT SELECT ON FUTURE TABLES` with a share. Every new or recreated object (`CREATE OR REPLACE`) must be **explicitly re-granted**, or consumers will lose access. |
| **No Object Limits, Only Structural Limits** | There is **no cap** on the number of tables or views in a share, as long as they all live inside that single shared database (or are bridged via Secure Views). |
| **Zero Consumer Storage, Compute Only** | Consumers pay **$0 for storage** on shared objects and only pay for the warehouse compute used to run `SELECT` queries — unless they're on a **Reader Account**, where **you (the provider)** pay for their compute. |
