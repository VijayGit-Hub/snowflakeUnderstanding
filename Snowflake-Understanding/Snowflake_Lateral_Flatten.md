# Snowflake `FLATTEN` & `LATERAL FLATTEN` Cheat Sheet

## 1. The Core Idea (In Plain English)
* **`FLATTEN`**: Takes a list (`ARRAY`) or key-value map (`OBJECT`) inside **one cell** and stretches it into **multiple rows**.
* **`LATERAL`**: A join keyword that lets a table function read columns from a table/function listed *before* it in the `FROM` clause. You need it when unpacking **nested** lists (a list inside a list).

---

## 2. Example 1: Basic `FLATTEN` (1 Level Deep)

### Before (`orders` table)

| order_id | items |
| :--- | :--- |
| 101 | `["Laptop", "Mouse"]` |
| 102 | `["Keyboard"]` |

### Query
```sql
SELECT 
  o.order_id, 
  f.index AS position,
  f.value::STRING AS item
FROM orders o,
TABLE(FLATTEN(INPUT => o.items)) f;
-- Note: For 1 level, TABLE(FLATTEN(...)) and LATERAL FLATTEN(...) work identically.
```

### After

| order_id | position | item |
| :--- | :--- | :--- |
| 101 | 0 | Laptop |
| 101 | 1 | Mouse |
| 102 | 0 | Keyboard |

---

## 3. Example 2: `LATERAL FLATTEN` (Nested Arrays / 2+ Levels Deep)

### Before (`users` table)

| user_name | purchases |
| :--- | :--- |
| Sam | `[{"store": "Target", "items": ["Milk", "Eggs"]}, {"store": "BestBuy", "items": ["TV"]}]` |

### Query
```sql
SELECT 
  u.user_name,
  p.value:store::STRING AS store,
  i.value::STRING AS item
FROM users u,
LATERAL FLATTEN(INPUT => u.purchases) p,   -- Step 1: Unpack the stores
LATERAL FLATTEN(INPUT => p.value:items) i; -- Step 2: Read 'p.value' from Step 1 and unpack items
```

### After

| user_name | store | item |
| :--- | :--- | :--- |
| Sam | Target | Milk |
| Sam | Target | Eggs |
| Sam | BestBuy | TV |

---

## 4. What `FLATTEN` Outputs Automatically
When you alias `FLATTEN(...) f`, Snowflake gives you 6 metadata columns:
* `f.value` — The actual element or value (most commonly used).
* `f.index` — The 0-based position if flattening an `ARRAY` (`0, 1, 2...`). `NULL` for objects.
* `f.key` — The key name if flattening an `OBJECT`. `NULL` for arrays.
* `f.path` — The path to the element within the data structure.
* `f.seq` — A unique sequence number tied to the input record.
* `f.this` — The original element being flattened.

---

## 5. Crucial Caveats & Gotchas (Don't Forget These!)

### Caveat 1: Empty or `NULL` Arrays Silently Drop Rows!
By default, `OUTER => FALSE`. If an order has `items = []` or `items = NULL`, **that entire order disappears from your result** (just like an `INNER JOIN`).
* **Fix:** Add `OUTER => TRUE` (acts like a `LEFT JOIN`, keeping the parent row with `NULL` values).
```sql
LATERAL FLATTEN(INPUT => o.items, OUTER => TRUE) f
```

### Caveat 2: Cartesian Explosion (Row Multiplication)
If you flatten **two independent arrays** at the same level in the same query (e.g., `tags` with 3 items and `items` with 4 items), Snowflake multiplies them (`3 × 4 = 12 rows` per order).
* **Fix:** Flatten independent arrays in separate CTEs and aggregate them before joining back.

### Caveat 3: Always Cast `f.value`
`f.value` returns a `VARIANT` data type. Even if it looks like `"Laptop"`, it still has JSON quotes around it until cast.
* **Fix:** Always cast explicitly: `f.value::STRING`, `f.value::INT`, `f.value::DATE`.

### Caveat 4: JSON Keys Are Case-Sensitive
In Snowflake SQL, column names are case-insensitive, **but JSON paths inside `VARIANT` are strictly case-sensitive**.
* `p.value:Store` will return `NULL` if the JSON key is `"store"`.

### Caveat 5: Flattening an `OBJECT` vs. an `ARRAY`
`FLATTEN` works on both arrays (`[...]`) and objects (`{"a": 1, "b": 2}`).
* On an **Array**: `f.index` has numbers, `f.key` is `NULL`.
* On an **Object**: `f.key` has `"a"`, `"b"`, and `f.index` is `NULL`.
* **Fix:** If a column might contain mixed types, enforce what you expect using `MODE => 'ARRAY'` or `MODE => 'OBJECT'`.

### Caveat 6: Using `PATH =>` Instead of Chaining
If you only need to reach a nested array inside objects (not an array inside an array), you don't need two `LATERAL FLATTEN`s—use `PATH`.
```sql
-- If payload is {"order": {"items": ["A", "B"]}}
LATERAL FLATTEN(INPUT => o.payload, PATH => 'order.items') f
```
