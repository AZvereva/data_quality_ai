# Data Quality Checks

For analytics engineers who want clean data without the noise.

---

## Schema Drift

**Problem:** Table structure changes without warning.  
**Consequences:** Your query selects columns that no longer exist, or misses new critical fields. Pipeline fails or produces incomplete data.  
**Decision:** Check schema before every transformation.

```sql
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'your_table'
ORDER BY ordinal_position;
```

---

## Volume Anomalies

**Problem:** Row counts change unexpectedly.  
**Consequences:** 50% drop = upstream pipeline broke. 300% spike = duplicate data. Zero rows = silent failure. All lead to wrong business decisions.  
**Decision:** Compare today's count to historical average. Stop if deviation > 20%.

```sql
SELECT 
    COUNT(*) as today_count,
    (SELECT AVG(daily_count) 
     FROM (SELECT COUNT(*) as daily_count 
           FROM your_table 
           WHERE created_at > CURRENT_DATE - INTERVAL '30 days'
           GROUP BY DATE(created_at)) sub) as avg_count,
    COUNT(*) / NULLIF((SELECT AVG(daily_count) FROM ...), 0) as ratio
FROM your_table
WHERE DATE(created_at) = CURRENT_DATE;
```

---

## Division by Zero

**Problem:** Denominator in calculation equals zero.  
**Consequences:** Query crashes. Dashboard shows error. Report to executives fails on delivery.  
**Decision:** Always guard division with NULLIF or CASE.

```sql
-- WRONG
SELECT revenue / orders as aov FROM metrics;

-- RIGHT
SELECT revenue / NULLIF(orders, 0) as aov FROM metrics;
-- Or: CASE WHEN orders = 0 THEN NULL ELSE revenue / orders END
```

---

## NULL Handling

**Problem:** Missing values treated as zero or ignored.  
**Consequences:** COALESCE(sales, 0) fabricates data. Summaries are wrong. Decisions based on imaginary numbers.  
**Decision:** Explicitly flag NULLs. Never silently convert to zero.

```sql
-- WRONG: Silent fabrication
SELECT COALESCE(sales, 0) as sales FROM table;

-- RIGHT: Explicit flag
SELECT 
    sales,
    CASE WHEN sales IS NULL THEN 'MISSING' ELSE 'OK' END as flag
FROM table;
```

---

## Duplicate Records

**Problem:** Same entity appears multiple times.  
**Consequences:** Revenue double-counted. Customer counts inflated. Metrics meaningless.  
**Decision:** Check primary key uniqueness before aggregation.

```sql
-- Primary key duplicates
SELECT id, COUNT(*) 
FROM your_table 
GROUP BY id 
HAVING COUNT(*) > 1;

-- Semantic duplicates (same email, different IDs)
SELECT email, COUNT(DISTINCT customer_id) 
FROM customers 
GROUP BY email 
HAVING COUNT(DISTINCT customer_id) > 1;
```

---

## Impossible Values

**Problem:** Data violates business reality.  
**Consequences:** Future dates in order history. Negative ages. 200-year-old customers. Analysis becomes joke.  
**Decision:** Define valid ranges. Reject or flag violations.

```sql
SELECT 
    CASE 
        WHEN order_date > CURRENT_DATE THEN 'FUTURE'
        WHEN order_date < '2000-01-01' THEN 'TOO_OLD'
        WHEN amount < 0 THEN 'NEGATIVE'
        ELSE 'OK'
    END as flag,
    COUNT(*)
FROM orders
GROUP BY flag
HAVING flag != 'OK';
```

---

## Orphan Records

**Problem:** Foreign keys point to non-existent parent records.  
**Consequences:** Joins silently drop data. Reports undercount. Relationships appear broken when they exist.  
**Decision:** Verify referential integrity before joins.

```sql
SELECT COUNT(*) as orphan_count
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL 
  AND o.customer_id IS NOT NULL;
```

---

## Post-Transform Reconciliation

**Problem:** Transformation changes totals unexpectedly.  
**Consequences:** Source says $1M revenue, dashboard shows $900K. $100K vanishes. No one trusts data.  
**Decision:** Match source totals after every transformation.

```sql
WITH source AS (SELECT SUM(amount) as total FROM raw_table),
     transformed AS (SELECT SUM(amount) as total FROM clean_table)
SELECT 
    s.total = t.total as match,
    ABS(s.total - t.total) as difference
FROM source s, transformed t;
```

---

## Data Freshness

**Problem:** Pipeline delivers stale data.  
**Consequences:** Decisions based on yesterday's data in real-time business. Missed opportunities. Wrong actions.  
**Decision:** Check last update time. Stop if > SLA threshold.

```sql
SELECT 
    MAX(updated_at) as last_update,
    EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - MAX(updated_at)))/3600 as hours_old
FROM your_table;
-- Decision: If hours_old > 4, alert and stop.
```

---

## Checklists

### Before Every Commit
- [ ] Schema unchanged
- [ ] Volume in expected range
- [ ] No division by zero
- [ ] NULLs flagged, not faked
- [ ] No duplicates
- [ ] No impossible values
- [ ] Source totals reconciled

### Weekly Review
- [ ] All freshness checks passing
- [ ] No new orphan records
- [ ] Distribution shapes unchanged
- [ ] Documentation updated with new edge cases

---

## Principle

Dirty data is the default. Clean data requires explicit decisions at every step.
