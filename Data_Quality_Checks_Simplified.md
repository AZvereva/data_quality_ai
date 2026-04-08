# Data Quality Checks — Quick Reference

For analytics engineers who need clean data without the noise.

---

## Table of Contents

- [Tools That Help (Besides SQL)](#tools-that-help-besides-sql)
- [Part 1: Incoming Data is Dirty](#part-1-incoming-data-is-dirty)
  - [Schema Drift](#schema-drift)
  - [Volume Anomalies](#volume-anomalies)
  - [Data Freshness](#data-freshness)
  - [Duplicate Records](#duplicate-records)
  - [Orphan Records](#orphan-records)
  - [Type Mismatches](#type-mismatches)
  - [Encoding & Character Issues](#encoding--character-issues)
  - [Whitespace & Padding](#whitespace--padding)
  - [Case Standardization](#case-standardization)
  - [Flexible Categorization](#flexible-categorization)
  - [Defensive Value Mapping](#defensive-value-mapping)
  - [Impossible Values](#impossible-values)
  - [Outliers](#outliers)
  - [Missing Values (NULLs)](#missing-values-nulls)
- [Part 2: Adjusting Data for Analysis](#part-2-adjusting-data-for-analysis)
  - [Division by Zero](#division-by-zero)
  - [Aggregation Errors](#aggregation-errors)
  - [Date Arithmetic Errors](#date-arithmetic-errors)
  - [Implicit Cast Failures](#implicit-cast-failures)
  - [Circular References](#circular-references)
  - [Post-Transform Reconciliation](#post-transform-reconciliation)
- [Part 3: Monitoring & Alerting](#part-3-monitoring--alerting)
  - [DWH Architecture for Quality](#dwh-architecture-for-quality)
  - [Monitoring Methods](#monitoring-methods)
  - [Alerting Channels](#alerting-channels)
  - [Checklists](#checklists)

---

## Tools That Help (Besides SQL)

Before diving into SQL fixes, know these tools exist to make your life easier:

| Tool | What It Does | When to Use |
|------|--------------|-------------|
| **dbt** | Runs SQL tests automatically, tracks data lineage | Daily pipeline checks, schema validation |
| **Great Expectations** | Pre-built quality rules (null checks, ranges, etc.) | Complex validation suites, documentation |
| **Airflow/Prefect** | Schedules your SQL checks, sends alerts | Orchestrating quality checks in pipelines |
| **BI Tools** (Tableau, Looker, Metabase) | Visual dashboards showing data health | Business stakeholder visibility |
| **Slack/Email alerts** | Notification when checks fail | Immediate response to issues |

**Bottom line:** SQL is your validation logic. These tools schedule, run, and alert on that logic.

---

## Part 1: Incoming Data is Dirty

Fix or flag problems before any transformation.

---

### Schema Drift

**The Problem**  
Table structure changes without warning—columns disappear or new ones appear.

**What Happens**  
Your query breaks because it selects a column that no longer exists. Or you miss critical new data.

**SQL Example & Fix**
```sql
-- Check current schema before running transformations
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'your_table'
ORDER BY ordinal_position;

-- Fix: Use explicit column lists, never SELECT *
-- Fix: Add schema validation in your pipeline
```

---

### Volume Anomalies

**The Problem**  
Row counts change unexpectedly—sudden drops or spikes.

**What Happens**  
50% drop = broken upstream pipeline. 300% spike = duplicate data. Zero rows = silent failure.

**SQL Example & Fix**
```sql
-- Check today's volume vs. yesterday
SELECT 
    COUNT(*) as today_count,
    LAG(COUNT(*)) OVER (ORDER BY CURRENT_DATE) as yesterday_count,
    COUNT(*) * 1.0 / NULLIF(LAG(COUNT(*)) OVER (ORDER BY CURRENT_DATE), 0) as ratio
FROM orders
WHERE order_date = CURRENT_DATE;

-- Fix: Stop pipeline if count differs >20% from average
```

---

### Data Freshness

**The Problem**  
Pipeline delivers stale data.

**What Happens**  
Decisions based on yesterday's data. Missed opportunities. Wrong actions.

**SQL Example & Fix**
```sql
-- Check how old the data is
SELECT 
    MAX(updated_at) as last_update,
    EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - MAX(updated_at)))/3600 as hours_old
FROM your_table;

-- Fix: Alert if hours_old > your SLA (e.g., 4 hours)
```

---

### Duplicate Records

**The Problem**  
Same entity appears multiple times.

**What Happens**  
Revenue double-counted. Customer counts inflated. Metrics meaningless.

**SQL Example & Fix**
```sql
-- Find duplicate primary keys
SELECT id, COUNT(*) 
FROM your_table 
GROUP BY id 
HAVING COUNT(*) > 1;

-- Fix: Deduplicate before aggregating
-- Select * will also return rn. So getting back to rule above - always be explicit in select statement about the fileds to return!
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) as rn
    FROM your_table
) WHERE rn = 1;
```

---

### Orphan Records

**The Problem**  
Foreign keys point to non-existent parent records.

**What Happens**  
Joins silently drop data. Reports undercount. Relationships appear broken.

**SQL Example & Fix**
```sql
-- Find orphans (orders without customers)
SELECT COUNT(*) as orphan_count
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL 
  AND o.customer_id IS NOT NULL;

-- Fix: Filter orphans or quarantine them for review
```

---

### Type Mismatches

**The Problem**  
Data arrives as wrong type (text instead of numbers, dates as strings).

**What Happens**  
Casting fails or produces wrong results. Sorting becomes alphabetical instead of numeric.

**SQL Example & Fix**
```sql
-- Validate before casting
SELECT 
    CASE 
        WHEN amount ~ '^[0-9]+\.?[0-9]*$' THEN amount::DECIMAL
        ELSE NULL
    END as amount_clean
FROM orders;

-- Fix: Clean invalid formats before casting
```

---

### Encoding & Character Issues

**The Problem**  
Special characters, invisible spaces, or encoding mismatches.

**What Happens**  
'Paris' ≠ 'Paris' (with invisible character). Group by produces multiple versions of same value.

**SQL Example & Fix**
```sql
-- Standardize text
SELECT 
    LOWER(TRIM(REGEXP_REPLACE(customer_name, '[^a-zA-Z0-9\s]', '', 'g'))) as clean_name
FROM customers;

-- Fix: Always clean text on ingestion
```

---

### Whitespace & Padding

**The Problem**  
Leading/trailing spaces or multiple internal spaces.

**What Happens**  
'ABC Corp' and 'ABC Corp ' don't match. Joins fail silently.

**SQL Example & Fix**
```sql
-- Find padded values
SELECT product_code
FROM products
WHERE product_code != TRIM(product_code);

-- Fix: Always TRIM() string fields
SELECT TRIM(BOTH ' ' FROM product_code) as clean_code FROM products;
```

---

### Case Standardization

**The Problem**  
Same value in different cases (USA, usa, Usa).

**What Happens**  
Group by splits single category into multiple rows.

**SQL Example & Fix**
```sql
-- Check case variations
SELECT country, COUNT(*) 
FROM customers 
GROUP BY country 
ORDER BY LOWER(country);

-- Fix: Standardize case
SELECT LOWER(country) as standard_country FROM customers;
```

---

### Flexible Categorization

**The Problem**  
Hardcoded CASE statements break when new values appear.

**What Happens**  
Uncategorized values become NULL. Reports show incomplete data.

**SQL Example & Fix**
```sql
-- WRONG: Silent NULLs for new values
SELECT 
    CASE region
        WHEN 'US' THEN 'North America'
        WHEN 'CA' THEN 'North America'
    END as region_group
FROM staging.customers;

-- RIGHT: Visible unmapped values
SELECT 
    CASE 
        WHEN region IN ('US', 'CA') THEN 'North America'
        WHEN region IN ('UK', 'DE') THEN 'Europe'
        ELSE 'UNMAPPED: ' || region  -- Shows what needs fixing
    END as region_group
FROM staging.customers;
```

---

### Defensive Value Mapping

**The Problem**  
Mapping tables miss target values, creating gaps.

**What Happens**  
'New York' in source becomes NULL because mapper only handled 'ny' and 'nyc'.

**SQL Example & Fix**
```sql
-- WRONG: Target not in list
SELECT 
    CASE 
        WHEN city = 'ny' THEN 'New York'
        WHEN city = 'nyc' THEN 'New York'
        -- 'New York' falls through to NULL!
    END as city_standardized
FROM staging.locations;

-- RIGHT: Include target spelling
SELECT 
    CASE 
        WHEN city IN ('ny', 'nyc', 'new york', 'New York') THEN 'New York'
        ELSE 'UNMAPPED: ' || city
    END as city_standardized
FROM staging.locations;
```

---

### Impossible Values

**The Problem**  
Data violates business reality (future dates, negative ages).

**What Happens**  
Analysis becomes unreliable. Executives lose trust in data.

**SQL Example & Fix**
```sql
-- Flag impossible values
SELECT 
    CASE 
        WHEN order_date > CURRENT_DATE THEN 'FUTURE'
        WHEN order_date < '2000-01-01' THEN 'TOO_OLD'
        WHEN amount < 0 THEN 'NEGATIVE'
        WHEN email NOT LIKE '%@%.%' THEN 'INVALID_EMAIL'
        ELSE 'OK'
    END as flag,
    COUNT(*)
FROM orders
GROUP BY flag
HAVING flag != 'OK';

-- Fix: Reject or quarantine flagged records
```

---

### Outliers

**The Problem**  
Values statistically valid but business-impossible (e.g., $1M order when max is $10K).

**What Happens**  
Means skewed. Aggregations misleading. Fraud or errors hidden.

**SQL Example & Fix**
```sql
-- Flag statistical outliers (beyond 3 standard deviations)
WITH stats AS (
    SELECT AVG(order_amount) as mean_val, STDDEV(order_amount) as std_val
    FROM orders
)
SELECT 
    o.order_id,
    o.order_amount,
    CASE 
        WHEN ABS(o.order_amount - s.mean_val) > 3 * s.std_val 
        THEN 'OUTLIER' ELSE 'OK'
    END as flag
FROM orders o
CROSS JOIN stats s;

-- Fix: Flag but don't remove—let business decide
```

---

### Missing Values (NULLs)

**The Problem**  
Data is absent.

**What Happens**  
Aggregations wrong. Joins drop rows. You cannot distinguish "zero" from "unknown."

**SQL Example & Fix**
```sql
-- WRONG: Silent conversion to zero
SELECT COALESCE(sales, 0) as sales FROM table;

-- RIGHT: Preserve numeric output with flag
SELECT 
    COALESCE(sales, 0) as sales_value,
    CASE WHEN sales IS NULL THEN 1 ELSE 0 END as is_imputed,
    CASE WHEN sales IS NULL THEN 'MISSING' ELSE 'OK' END as quality_flag
FROM table;
```

---

## Part 2: Adjusting Data for Analysis

Prevent errors in your own transformations.

---

### Division by Zero

**The Problem**  
Denominator in calculation equals zero.

**What Happens**  
Query crashes. Dashboard shows error. Report to executives fails.

**SQL Example & Fix**
```sql
-- WRONG
SELECT revenue / orders as aov FROM metrics;

-- RIGHT
SELECT revenue / NULLIF(orders, 0) as aov FROM metrics;
-- Or: CASE WHEN orders = 0 THEN NULL ELSE revenue / orders END
```

---

### Aggregation Errors

**The Problem**  
SUM, AVG, COUNT on dirty data produces wrong results.

**What Happens**  
Double-counting from duplicates. Averages skewed by outliers.

**SQL Example & Fix**
```sql
-- Check for duplicates before aggregating
SELECT customer_id, order_date, amount, COUNT(*) as occurrences
FROM orders
GROUP BY customer_id, order_date, amount
HAVING COUNT(*) > 1;

-- Fix: Use COUNT(DISTINCT) for unique counts
-- Fix: Clean data before aggregating
```

---

### Date Arithmetic Errors

**The Problem**  
Calculating intervals, ages, durations with NULL or invalid dates.

**What Happens**  
Negative ages. Division by zero in duration calculations. Wrong cohort analysis.

**SQL Example & Fix**
```sql
-- Calculate age safely
SELECT 
    customer_id,
    birth_date,
    CASE 
        WHEN birth_date IS NULL THEN NULL
        WHEN birth_date > CURRENT_DATE THEN NULL
        ELSE DATE_PART('year', AGE(CURRENT_DATE, birth_date))
    END as age_years,
    CASE 
        WHEN birth_date IS NULL THEN 'MISSING_BIRTH_DATE'
        WHEN birth_date > CURRENT_DATE THEN 'FUTURE_DATE'
        ELSE 'OK'
    END as flag
FROM customers;
```

---

### Implicit Cast Failures

**The Problem**  
Database implicitly casts types, producing unexpected results.

**What Happens**  
'2024-01-01' > '2024-1-1' (string comparison). 5 / 2 = 2 (integer division).

**SQL Example & Fix**
```sql
-- WRONG: Implicit string comparison
SELECT * FROM orders WHERE order_date > '2024-01-01';

-- RIGHT: Ensure column is DATE type
-- Cast only in final output, not in filters
SELECT 
    order_id,
    amount::DECIMAL(12,2) as amount_decimal
FROM orders
WHERE order_date > '2024-01-01';
```

---

### Circular References

**The Problem**  
Self-referencing calculations or recursive CTEs that never terminate.

**What Happens**  
Query hangs. Resources exhausted. Database locks.

**SQL Example & Fix**
```sql
-- Safe recursive CTE with limit and cycle detection
WITH RECURSIVE hierarchy AS (
    SELECT employee_id, manager_id, 1 as level,
           ARRAY[employee_id] as visited
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id, h.level + 1,
           h.visited || e.employee_id
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.employee_id
    WHERE h.level < 10  -- Prevent deep recursion
      AND NOT e.employee_id = ANY(h.visited)  -- Prevent cycles
)
SELECT * FROM hierarchy;
```

---

### Post-Transform Reconciliation

**The Problem**  
Transformation changes totals unexpectedly.

**What Happens**  
Source says $1M revenue, dashboard shows $900K. $100K vanishes. No one trusts data.

**SQL Example & Fix**
```sql
-- Reconcile source vs. transformed totals
WITH source AS (SELECT SUM(amount) as total FROM raw_table),
     transformed AS (SELECT SUM(amount) as total FROM clean_table)
SELECT 
    s.total = t.total as match,
    ABS(s.total - t.total) as difference,
    ABS(s.total - t.total) / NULLIF(s.total, 0) * 100 as pct_variance
FROM source s, transformed t;

-- Fix: Investigate if variance > 0.1%
```

---

## Part 3: Monitoring & Alerting

Systematic observation of data health.

---

### DWH Architecture for Quality

**Three-Layer Quality Architecture:**

| Layer | Purpose | Quality Mechanism |
|-------|---------|-------------------|
| **Staging** | Raw data landing | Schema validation, type checking, quarantine bad records |
| **Intermediate** | Cleaned/transformed | Referential integrity, business rules, reconciliation |
| **Mart** | Business-ready | Metric validation, semantic checks, drift detection |

**Quarantine Pattern:** Separate invalid records from valid ones, allowing pipeline to continue while isolating problems.

---

### Monitoring Methods

**1. Table Constraints (Hard Enforcement)**
```sql
-- Enforce rules at database level
ALTER TABLE orders ADD CONSTRAINT chk_positive_amount CHECK (amount >= 0);
ALTER TABLE customers ADD PRIMARY KEY (customer_id);
```

**2. dbt Tests (Soft Enforcement)**
```yaml
# Run these automatically in your pipeline
columns:
  - name: order_id
    tests:
      - unique
      - not_null
  - name: amount
    tests:
      - dbt_utils.accepted_range:
          min_value: 0
```

**3. Separate QA Tables (Audit Trail)**
```sql
-- Log quality metrics over time
CREATE TABLE audit.daily_quality_metrics (
    check_date DATE,
    table_name VARCHAR(100),
    check_name VARCHAR(100),
    metric_value NUMERIC,
    status VARCHAR(20)
);
```

**4. BI Dashboard Monitoring**
- Health score by table (0-100%)
- Trend lines showing quality over time
- Top failing checks

---

### Alerting Channels

| Severity | Channel | Response Time | Examples |
|----------|---------|---------------|----------|
| **CRITICAL** | PagerDuty + Slack @channel | 15 minutes | Pipeline failure, freshness > 8hrs |
| **HIGH** | Slack #data-alerts | 1 hour | Schema change, volume anomaly > 50% |
| **MEDIUM** | Slack #data-quality | 4 hours | dbt test failure, outlier spike |
| **LOW** | Email digest | 24 hours | Minor threshold breach |

---

### Checklists

**Before Every Commit**
- [ ] Volume in expected range (±20%)
- [ ] No duplicates in source
- [ ] No orphans in joins
- [ ] Whitespace trimmed
- [ ] Case standardized
- [ ] No impossible values
- [ ] NULLs handled with flags (not converted to zero)
- [ ] No division by zero
- [ ] Source totals reconciled (variance < 0.1%)

**Weekly Review**
- [ ] All freshness checks passing
- [ ] No new orphan records
- [ ] Data health score > 95%
- [ ] Alert volume reviewed

---

*Simplified version for quick reference. For detailed implementation, see the full guide.*
