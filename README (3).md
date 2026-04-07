#Hands-on L11: AWS Core Services — S3, Glue, CloudWatch & Athena

Name:Vamsi Aakash Samudrala
Student id : 801425922
Email: vsamudr2@charlotte.edu

> Course: ITCS 6190 — Cloud Computing for Data Analysis  
> Dataset: [Amazon E-Commerce Sales Data (Kaggle)](https://www.kaggle.com/datasets/thedevastator/unlock-profits-with-e-commerce-sales-data)

---

##  Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Workflow](#architecture--workflow)
3. [Step-by-Step Setup](#step-by-step-setup)
   - [Step 1 — Amazon S3 Bucket Setup](#step-1--amazon-s3-bucket-setup)
   - [Step 2 — IAM Role Configuration](#step-2--iam-role-configuration)
   - [Step 3 — AWS Glue Crawler](#step-3--aws-glue-crawler)
   - [Step 4 — CloudWatch Monitoring](#step-4--cloudwatch-monitoring)
   - [Step 5 — Amazon Athena Query Editor](#step-5--amazon-athena-query-editor)
4. [SQL Queries & Results](#sql-queries--results)
   - [Query 1 — Basic Table Exploration](#query-1--basic-table-exploration)
   - [Query 2 — Orders by Product Category](#query-2--orders-by-product-category)
   - [Query 3 — Revenue by Fulfilment Method](#query-3--revenue-by-fulfilment-method)
   - [Query 4 — Monthly Sales Trend](#query-4--monthly-sales-trend)
   - [Query 5 — Top 5 Best-Selling SKUs per Category](#query-5--top-5-best-selling-skus-per-category)
5. [Key Findings](#key-findings)
6. [Repository Structure](#repository-structure)
7. [Challenges Faced](#challenges-faced)

---

## Project Overview

This hands-on demonstrates how to build a **serverless data analytics pipeline** on AWS by chaining together four managed services:

| AWS Service | Role in Pipeline |
|---|---|
| Amazon S3 | Raw data lake — stores the uploaded CSV dataset |
| AWS IAM | Security — grants Glue the permissions needed to read S3 |
| AWS Glue | ETL / Cataloging — crawls S3 and creates a queryable table schema |
| Amazon CloudWatch | Observability — monitors and logs the Glue crawler run |
| Amazon Athena | Analytics — runs standard SQL directly against S3 data via the Glue catalog |

No servers, no databases to provision — just upload data and query.

---

## Architecture & Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Analytics Pipeline                       │
│                                                                     │
│   CSV Dataset        Amazon S3         AWS Glue         Amazon      │
│   (Kaggle)    ──▶   (raw/ folder) ──▶  Crawler   ──▶   Athena      │
│                                            │                │       │
│                                       CloudWatch        SQL Query   │
│                                        (Logs)           Results     │
└─────────────────────────────────────────────────────────────────────┘
```

Workflow Steps:
1. Configure Amazon S3 buckets (`raw/` for input, `process/` for output)
2. Create an IAM role granting Glue access to S3
3. Create and run a Glue crawler to auto-discover the schema
4. Monitor crawler execution via CloudWatch logs
5. Query the catalogued data using Athena SQL editor

---

## Step-by-Step Setup

### Step 1 — Amazon S3 Bucket Setup

**Approach:** Amazon S3 (Simple Storage Service) serves as the data lake for this pipeline. Two folders are created inside one bucket to separate raw ingested data from any processed outputs.

What was done:
- Created a general-purpose S3 bucket named **`handsonn11cloudcomputingg`** in the `us-east-1` (N. Virginia) region
- Created two folders inside the bucket:
  - `raw/` — where the Amazon sales CSV dataset is uploaded
  - `process/` — reserved for processed/transformed output data

**Why this matters:** Glue needs a well-defined S3 path to point its crawler at. The `raw/` prefix acts as the data source, and separating it from `process/` is a best-practice pattern for data lake organization.



> Bucket: `handsonn11cloudcomputingg` | **Region:** us-east-1 | **Objects:** 2 folders (`process/`, `raw/`)
<img width="1710" height="1107" alt="Amazon_S3" src="https://github.com/user-attachments/assets/b53bb8b0-a7cd-4dd2-92a5-d5111f6f3a2e" />

---

### Step 2 — IAM Role Configuration

**Approach:** AWS Identity and Access Management (IAM) is used to create a dedicated role for the Glue service. Rather than using long-lived credentials, IAM roles provide temporary, scoped permissions — following the principle of least privilege.

**What was done:**
- Navigated to IAM → Roles → Create Role
- Selected **AWS Glue** as the trusted entity (service principal)
- Attached policies granting S3 read access and Glue catalog write access
- Named the role **`GlueRole`**

Why this matters: Without this role, the Glue crawler cannot read objects from S3 or write table metadata to the Glue Data Catalog. The role is assigned directly to the crawler in the next step.


<img width="1710" height="1107" alt="IAM_Roles" src="https://github.com/user-attachments/assets/4a63e9cd-cad6-430e-aba8-eb826b6a2ef8" />

> Roles visible:`GlueRole` (AWS Service: glue) — Last activity: 6 hours ago. Three other auto-generated AWS service roles are also listed.

---

### Step 3 — AWS Glue Crawler

**Approach:** AWS Glue Crawler automatically inspects the data in S3, infers the schema (column names, data types), and registers a table in the Glue Data Catalog. This eliminates the need to manually define table schemas.

What was done:
- Created a crawler named **`Crawlerrr`** in AWS Glue Studio
- Configured the data source as the `raw/` folder in the S3 bucket
- Assigned the `GlueRole` IAM role created in Step 2
- Set the output database to **`outputgluedatabase`**
- Ran the crawler — it completed in ~40–43 seconds across two runs
- Both runs reported **"1 table change, 0 partition changes"** confirming the `raw` table was created/updated

Why this matters: The crawler bridges S3 and Athena. Without it, Athena has no schema to work with and cannot interpret the CSV file as a structured table.

<img width="1710" height="1107" alt="AWS_Glue_Crawler" src="https://github.com/user-attachments/assets/48606bde-182b-4e4c-a2d0-913388011db6" />


> Crawler: `Crawlerrr` | **State:** READY | **Database:** `outputgluedatabase` | **IAM Role:** GlueRole  
> Runs:2 completed — April 7, 2026 at 15:56 and 16:11 UTC

---

### Step 4 — CloudWatch Monitoring

Approach: Amazon CloudWatch captures all log events emitted by the Glue crawler during its run. This provides full observability into what the crawler did — from start to finish — without needing to SSH into any server.

What was done:
- Navigated from the Glue Crawler page → "View CloudWatch logs"
- Filtered logs by the crawler run ID: `7237fecf-a595-4e8d-b730-b282bc1342f8`
- Observed the complete lifecycle of the crawler run in chronological order

Why this matters:** CloudWatch confirms the crawler successfully classified the data, created the `raw` table in the Glue catalog, and terminated cleanly — essential for debugging if anything goes wrong.


<img width="1710" height="1107" alt="CloudWatch" src="https://github.com/user-attachments/assets/a54cb884-7089-4a90-a4d4-6ba44efeca75" />

> Log Group: `/aws-glue/crawlers` | **Log Stream:** `Crawlerrr` | **Result:** Table `raw` added to `outputgluedatabase`

---

### Step 5 — Amazon Athena Query Editor

Approach: Amazon Athena is a serverless interactive query service that uses standard ANSI SQL to query data directly in S3 through the Glue Data Catalog. There is no need to load data into a separate database — Athena queries the CSV files in-place.

What was done:
- Opened Athena Query Editor in the AWS Console
- Set the data source to **`AwsDataCatalog`** and the database to **`outputgluedatabase`**
- Confirmed two tables available: `amazon_sale_report_csv` and `raw`
- Set the workgroup to **`primary`** and configured an S3 query results location
- Ran 5 analytical queries (detailed below)

Why this matters:Athena provides on-demand SQL analytics at scale with no infrastructure management. Charges are per query based on data scanned — efficient partitioning and filtering directly reduce cost.
<img width="1710" height="1107" alt="Athena" src="https://github.com/user-attachments/assets/ce49cdc0-3678-4413-b961-66e0c22748d3" />



> Data Source: AwsDataCatalog | **Database:** outputgluedatabase | **Tables:** `raw`, `amazon_sale_report_csv`

---

## SQL Queries & Results

All queries run against the `outputgluedatabase.raw` table. Cancelled and pending orders are excluded from revenue/fulfilment queries using:
```sql
WHERE LOWER(status) NOT IN ('cancelled', 'pending', 'pending - waiting for pick up')
```

---

### Query 1 — Basic Table Exploration

Objective: Retrieve the first 10 records to understand the dataset structure — column names, data types, and sample values.

**Approach: A simple `SELECT *` with `LIMIT 10` to preview the raw data before writing analytical queries. This is always the first step in any data exploration workflow.

```sql
SELECT *
FROM raw
LIMIT 10;
```

**Query Screenshot:**
<img width="1710" height="1107" alt="Q1" src="https://github.com/user-attachments/assets/4e1255f5-08df-4441-955d-04eabf690736" />



**Results Screenshot:**
<img width="1710" height="1107" alt="Q1_Result" src="https://github.com/user-attachments/assets/a0332c58-e0bf-4655-b2db-08cd588b52f0" />



**Findings:** The dataset contains columns including `index`, `order id`, `date`, `status`, `fulfilment`, `sales-channel`, `ship-service-level`, `style`, `sku`, `category`, `size`, `asin`, `courier status`, `qty`, `currency`, `amount`, `ship-city`, `ship-state`, `ship-postal-code`, `ship-country`, and `b2b`. Data is from April–June 2022, Amazon India marketplace. Orders have statuses like Shipped, Cancelled, and "Shipped - Delivered to Buyer".

> ⏱ **Run time:** 711 ms | **Data scanned:** 1.54 MB | **Results:** 10 rows

---

### Query 2 — Orders by Product Category

**Objective:** Count how many orders exist per product category to understand which categories are most popular.

**Approach:** Aggregate by `category` using `COUNT(*)`, then sort descending by `total_orders` to surface the top-selling categories first. No status filter is applied here — all orders including cancelled are counted to reflect total demand volume.

```sql
SELECT
    category,
    COUNT(*) AS total_orders
FROM raw
GROUP BY category
ORDER BY total_orders DESC
LIMIT 10;
```

**Query Screenshot:**
<img width="1710" height="1107" alt="Q2" src="https://github.com/user-attachments/assets/ea7a6554-ea45-4117-9c6e-8cf36db3417f" />



**Results Screenshot:**

<img width="1710" height="1107" alt="Q2_Result" src="https://github.com/user-attachments/assets/a9d24b4d-a4ca-4b36-8e3e-ae604af723e0" />


**Findings:** 9 product categories found. **Set** leads with 50,284 orders, followed by **Kurta** (49,877) and **Western Dress** (15,500). At the bottom are **Saree** (164) and **Dupatta** (3). Sets and Kurtas together account for ~75% of all orders, indicating these are the core product lines.

> ⏱ **Run time:** 1.112 sec | **Data scanned:** 65.73 MB | **Results:** 9 rows

---

### Query 3 — Revenue and Quantity by Fulfilment Method

**Objective:** Compare Amazon-fulfilled vs. Merchant-fulfilled orders across total orders, units sold, and revenue — excluding non-completed orders.

**Approach:** Filter out cancelled and pending statuses to focus only on completed/shipped transactions. Group by `fulfilment` and aggregate revenue using `ROUND(SUM(amount), 2)`. Sort by highest revenue first.

```sql
SELECT
    fulfilment,
    COUNT(*)               AS total_orders,
    SUM(qty)               AS total_units_sold,
    ROUND(SUM(amount), 2)  AS total_revenue
FROM raw
WHERE LOWER(status) NOT IN ('cancelled', 'pending', 'pending - waiting for pick up')
GROUP BY fulfilment
ORDER BY total_revenue DESC
LIMIT 10;
```

**Query Screenshot:**

<img width="1710" height="1107" alt="Q3" src="https://github.com/user-attachments/assets/19a7c100-a9ea-4bf8-b865-dd1068b9f996" />


**Results Screenshot:**

<img width="1710" height="1107" alt="Q3_Result" src="https://github.com/user-attachments/assets/f53e8e77-b1d3-4b72-be5e-9d3b23c696a0" />


**Findings:** Amazon fulfillment dominates with **77,812 orders** generating **~$50.3M** in revenue (78,017 units), while Merchant fulfillment handled **31,892 orders** generating **~$20.7M** (32,035 units). Amazon-fulfilled orders produce roughly **2.4× the revenue** of merchant-fulfilled — suggesting either higher-value products or greater volume through the Amazon channel.

> ⏱ **Run time:** 1.008 sec | **Data scanned:** 65.73 MB | **Results:** 2 rows

---

### Query 4 — Monthly Sales Trend

**Objective:** Analyze how order volume and revenue evolved month over month, excluding non-completed orders, sorted chronologically.

**Approach:** The `date` column is a string in `MM-DD-YY` format, so it must be parsed using `DATE_PARSE(date, '%m-%d-%y')` before truncating to month granularity with `DATE_TRUNC('month', ...)`. Results are sorted chronologically (ASC) to observe the time trend.

```sql
SELECT
    DATE_TRUNC('month', DATE_PARSE(date, '%m-%d-%y'))  AS sales_month,
    COUNT(*)                                            AS total_orders,
    ROUND(SUM(amount), 2)                               AS total_revenue
FROM raw
WHERE LOWER(status) NOT IN ('cancelled', 'pending', 'pending - waiting for pick up')
GROUP BY DATE_TRUNC('month', DATE_PARSE(date, '%m-%d-%y'))
ORDER BY sales_month ASC
LIMIT 10;
```

Query Screenshot:
<img width="1710" height="1107" alt="Q4" src="https://github.com/user-attachments/assets/e3ef92e3-1efd-4a3d-9f49-395e079297c8" />


Results Screenshot:
<img width="1710" height="1107" alt="Q4_Result" src="https://github.com/user-attachments/assets/5d115d60-269f-4edb-9d81-60f73e505fd4" />



Findings:The dataset spans 4 months (March–June 2022). March 2022 had only **153 orders** ($94,810 revenue) — likely a partial month of data. April 2022 was the peak with **41,929 orders** (~$26.2M revenue). May (36,158 orders, ~$23.9M) and June (31,464 orders, ~$20.8M) show a declining trend, possibly reflecting post-peak seasonal correction.

> ⏱ **Run time:** 912 ms | **Data scanned:** 65.73 MB | **Results:** 4 rows

---

### Query 5 — Top 5 Best-Selling SKUs per Category

**Objective:** Identify the top 5 revenue-generating SKUs within each product category, using a window function to rank within partitions.

**Approach:** Uses a **Common Table Expression (CTE)** with `ROW_NUMBER() OVER (PARTITION BY category ORDER BY SUM(amount) DESC)` to rank SKUs by revenue within each category. The outer query filters to only `rnk <= 5`. Zero-quantity orders are excluded with `qty > 0` to avoid counting returns or data errors.

```sql
WITH sku_revenue AS (
    SELECT
        category,
        sku,
        ROUND(SUM(amount), 2)  AS total_revenue,
        SUM(qty)               AS total_units_sold,
        ROW_NUMBER() OVER (
            PARTITION BY category
            ORDER BY SUM(amount) DESC
        )                      AS rnk
    FROM raw
    WHERE LOWER(status) NOT IN ('cancelled', 'pending', 'pending - waiting for pick up')
      AND qty > 0
    GROUP BY category, sku
)
SELECT *
FROM sku_revenue
WHERE rnk <= 5
ORDER BY category, rnk
LIMIT 10;
```
Query Screenshot:

<img width="1710" height="1107" alt="Q5" src="https://github.com/user-attachments/assets/1fbc85d6-207a-490e-9924-21e7c8e51584" />


**Results Screenshot:**

<img width="1710" height="1107" alt="Q5_Result" src="https://github.com/user-attachments/assets/270cb63a-f1e3-476b-b4a4-335f902a85a8" />


Findings:For Blouse** category, SKU `J0217-BL-L` ranks #1 with $20,275 revenue (30 units sold). For **Bottom** category, SKU `BTM031-NP-XL` leads with $2,590 revenue (5 units). Blouse SKUs consistently generate higher per-SKU revenues than Bottom SKUs. The window function approach ensures each category has its own independent ranking — a standard pattern for top-N-per-group analytics.
|

## Repository Structure

```
📁 repository/
│
├── 📄 README.md                     ← This file
│
├── 📁 queries/
│   ├── Q1_basic_exploration.sql
│   ├── Q2_orders_by_category.sql
│   ├── Q3_revenue_by_fulfilment.sql
│   ├── Q4_monthly_sales_trend.sql
│   └── Q5_top_skus_per_category.sql
│
├── 📁 results/
│   ├── Q1_Result.csv
│   ├── Q2_Result.csv
│   ├── Q3_Result.csv
│   ├── Q4_Result.csv
│   └── Q5_Result.csv
│
└── 📁 screenshots/
    ├── Amazon_S3.png
    ├── IAM_Roles.png
    ├── AWS_Glue_Crawler.png
    ├── CloudWatch.png
    ├── Athena.png
    ├── Q1.png  /  Q1_Result.png
    ├── Q2.png  /  Q2_Result.png
    ├── Q3.png  /  Q3_Result.png
    ├── Q4.png  /  Q4_Result.png
    └── Q5.png  /  Q5_Result.png
```

---

##Challenges Faced

### Challenge 1 — Athena Query Results Location Not Configured
Problem:When first opening Athena, running any query immediately threw an error: *"Query result location not set."* Athena requires an S3 path to write query results before it can execute anything.

### Challenge 2 — Date Column Stored as String, Not DATE Type
Problem: The `date` column in the dataset (format `MM-DD-YY`, e.g., `04-30-22`) was ingested by the Glue crawler as a plain `string` type. Athena does not automatically cast string columns to dates, so using standard date functions like `DATE_TRUNC` directly on the column caused a type mismatch error.
---
### Challenge 3 — Glue Crawler Ran Twice Due to Schema Mismatch

### Challenge 4 — Revenue Values Displayed in Scientific Notation
**Problem: In Query 3 and Query 4 results, the `total_revenue` column was displayed in scientific notation (e.g., `5.0324255E7`) in the Athena UI, making it difficult to read at a glance.

### Challenge 5 — Filtering Pending Orders Required LOWER() Normalization
Problem: The `status` column contained inconsistent casing and values. A simple `WHERE status != 'Cancelled'` filter was insufficient because some records had `'cancelled'` (lowercase) or variations like `'Pending - Waiting for Pick Up'` with mixed casing.

### Challenge 6 — IAM Role Permissions Had to Be Correctly Scoped
Problem:The initial attempt to run the Glue crawler failed because the IAM role did not have sufficient permissions to write metadata to the Glue Data Catalog, even though S3 read access was in place.



