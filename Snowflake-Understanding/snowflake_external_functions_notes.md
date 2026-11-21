# Snowflake External Functions — Complete Revision Notes

> **Quick Summary:** An **External Function** lets Snowflake run code that lives *outside* Snowflake (like on AWS Lambda) and return the result inside a normal SQL query. It uses an **API Integration** object to store security and authentication details.

---

## 1. Key Concept & Security Object

* **What is an External Function?**
  It is a database function that calls code executed outside Snowflake on a remote cloud service (AWS Lambda, Azure Functions, or Google Cloud Functions). Instead of running logic inside the database warehouse, Snowflake sends data over HTTPS to an external API endpoint and brings back the answer.
* **Which object holds security information?**
  **`API INTEGRATION`** (not `STORAGE INTEGRATION`).
  * **`API INTEGRATION`:** Stores IAM role ARNs, allowed API Gateway URLs, and authentication details for calling external functions/APIs.
  * **`STORAGE INTEGRATION`:** Used only for accessing external cloud storage buckets (like AWS S3, Azure Blob, or GCS) for loading/unloading data.

---

## 2. Why Do We Need External Functions?

Databases like Snowflake are locked down for security and cannot directly connect to the open internet. An external function acts as a secure bridge to the outside world.

**Common Real-World Uses:**
1. **Live Data Enrichment:** Calling third-party APIs during a SQL query to get live currency exchange rates, validate emails, or geocode addresses.
2. **External Libraries & Languages:** Running custom code or specialized libraries that are not natively supported inside Snowflake.
3. **Machine Learning Inference:** Sending rows from a table to an externally hosted ML model to get fraud scores or sentiment predictions.
4. **Custom Encryption / Tokenization:** Integrating with an external security vault to mask or unmask sensitive customer data.

---

## 3. Real-World Example: Live Currency Conversion (Snowflake + AWS Lambda)

### The Business Problem
* You have a `DAILY_ORDERS` table in Snowflake with sales recorded in different currencies (`USD`, `EUR`, `JPY`).
* Your finance team wants a SQL report converting every order into **INR** using **today's live exchange rate**.
* **Why SQL alone fails:** Snowflake does not know live exchange rates and cannot browse the internet to fetch them.
* **The Solution:** Build a Snowflake external function `convert_to_inr(amount, currency)` backed by an **AWS Lambda** function.

### How Data Flows (Step-by-Step)
1. **Snowflake** runs your `SELECT` query and packages table rows into a JSON payload.
2. **API Integration** provides the AWS IAM credentials so AWS trusts Snowflake.
3. **AWS API Gateway** acts as the front door, receiving the HTTPS request from Snowflake and forwarding it to Lambda.
4. **AWS Lambda (Python)** unpacks the rows, calls a live exchange-rate API, calculates the INR value, and formats the response.
5. **Snowflake** receives the JSON response and displays the converted numbers as a normal column in your query output.

---

## 4. Complete Implementation Code

### Step 1: AWS Lambda Function (Python)
Snowflake sends rows as a list of lists where index `0` is the row number (`[row_index, arg1, arg2]`). Lambda must return the result with the exact same `row_index` so Snowflake can match answers back to the right rows.

```python
import json
import urllib.request

def lambda_handler(event, context):
    # 1. Extract the rows sent by Snowflake
    body = json.loads(event['body'])
    rows = body['data']
    
    # 2. Fetch live exchange rates from a public API (Base currency = INR)
    url = "https://api.exchangerate-api.com/v4/latest/INR"
    response = urllib.request.urlopen(url)
    rates = json.loads(response.read())['rates']
    
    results = []
    
    # 3. Loop through each row:
    #    row[0] = row index (e.g., 0, 1, 2)
    #    row[1] = order amount (e.g., 50.00)
    #    row[2] = currency code (e.g., 'USD')
    for row in rows:
        row_index = row[0]
        amount = row[1]
        currency = row[2]
        
        # Convert foreign amount to INR
        inr_value = round(amount / rates[currency], 2)
        results.append([row_index, inr_value])
        
    # 4. Return data in the exact JSON format Snowflake expects
    return {
        'statusCode': 200,
        'body': json.dumps({'data': results})
    }
```

#### Why did we divide (`amount / rates[currency]`) instead of multiplying?
* Because the API endpoint (`/latest/INR`) uses **INR as the base currency**.
* It tells you how much foreign currency **1 INR** buys (`1 INR = 0.0118 USD`).
* To convert **$50 USD** into INR: `50 / 0.0118 = 4237.29 INR`.
* *(If your API used USD as the base currency where `1 USD = 84.50 INR`, you would multiply: `amount * rates['INR']`.)*

---

### Step 2: Snowflake Setup (SQL)

```sql
-- 1. Create the API Integration (holds AWS IAM Role & allowed URL prefix)
CREATE OR REPLACE API INTEGRATION aws_currency_api
  API_PROVIDER = aws_api_gateway
  API_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-lambda-role'
  API_ALLOWED_PREFIXES = ('https://xyz123.execute-api.ap-south-1.amazonaws.com/prod/')
  ENABLED = TRUE;

-- 2. Create the External Function pointing to your API Gateway endpoint
CREATE OR REPLACE EXTERNAL FUNCTION convert_to_inr(amount FLOAT, currency VARCHAR)
  RETURNS FLOAT
  API_INTEGRATION = aws_currency_api
  MAX_BATCH_ROWS = 500
  AS 'https://xyz123.execute-api.ap-south-1.amazonaws.com/prod/convert';
```

---

### Step 3: Calling the Function in SQL

```sql
SELECT 
    order_id,
    amount,
    currency,
    convert_to_inr(amount, currency) AS amount_in_inr
FROM daily_orders;
```

**Example Output:**

| ORDER_ID | AMOUNT | CURRENCY | AMOUNT_IN_INR |
| :--- | :--- | :--- | :--- |
| 101 | 100.00 | USD | 8450.00 |
| 102 | 50.00 | EUR | 4510.50 |
| 103 | 10000 | JPY | 5520.00 |

---

## 5. Batching & Controlling Rows (`MAX_BATCH_ROWS`)

### How Snowflake Handles Large Tables
If you query 10,000 rows, Snowflake **does not** make 10,000 separate HTTP calls to AWS Lambda. Instead, it automatically bundles thousands of rows into a single JSON batch request for speed and efficiency.

### How to Control Batch Size Yourself
You can set an upper limit on how many rows Snowflake sends per request using the **`MAX_BATCH_ROWS`** parameter:

```sql
MAX_BATCH_ROWS = 500
```

**Why limit batch size?**
1. **Prevent Timeouts:** AWS API Gateway has a hard timeout of **29 seconds**. Smaller batches ensure Lambda finishes processing before timing out.
2. **Respect External API Rate Limits:** If Lambda calls a third-party API with strict requests-per-second limits, smaller batches prevent throttling.
3. **Avoid Memory / Payload Limits:** Prevents hitting AWS Lambda's **6 MB** payload limit when rows contain large text or JSON strings.

**Key Things to Remember:**
* `MAX_BATCH_ROWS` is an **upper cap**, not a fixed size. If `MAX_BATCH_ROWS = 500` and your table only has 40 rows, Snowflake sends a batch of 40.
* Setting it too low (e.g., `1`) causes too many network round-trips, making queries slow and expensive.

---

## 6. Quick Exam & Interview Recap
* **Security Object:** `API INTEGRATION` (never `STORAGE INTEGRATION`).
* **Architecture:** Snowflake -> API Integration -> Cloud Proxy (AWS API Gateway) -> Remote Compute (AWS Lambda).
* **Row Matching:** Snowflake sends `[row_index, inputs...]` and expects `[row_index, output]` back in JSON.
* **Batch Control:** Use `MAX_BATCH_ROWS` in `CREATE EXTERNAL FUNCTION` to cap rows per HTTP call.
