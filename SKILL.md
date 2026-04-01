---
name: synthetic-data-bigquery
description: "Use this skill whenever the user wants to generate realistic synthetic data using BigQuery SQL. Triggers: 'generate synthetic data', 'create fake data', 'simulate user events', 'populate a dataset', 'mock data for BigQuery', 'realistic test data', 'agentic training data', 'synthetic trajectories', 'SQL data generation'. This skill produces ready-to-run BigQuery SQL that generates relational, temporally consistent, entity-coherent synthetic datasets for ML training, agentic AI, analytics testing, and data pipeline validation. Use it when the user needs multi-table environments, tool-use trajectories, chain-of-thought traces, or event logs grounded in realistic distributions. Do NOT use for CSV exports, Pandas DataFrames, or local SQLite generation unless BigQuery SQL is also requested."
license: Proprietary. LICENSE.txt has complete terms
---

# Synthetic Data Engineering — BigQuery SQL Playbook

## 🧭 Skill Philosophy

Realistic synthetic data is NOT random data. It must satisfy four hard constraints:

| Constraint | Meaning |
|---|---|
| **Entity Coherence** | The same entity (user, order, product) has identical attributes across all tables |
| **Temporal Causality** | Effects never precede causes. A refund timestamp > purchase timestamp. Always. |
| **Statistical Realism** | Distributions match the domain (power-law user activity, Gaussian prices, Zipf product popularity) |
| **Relational Integrity** | Foreign keys resolve. No orphan rows. Join graphs are sound. |

Always read the full task before writing SQL. Identify: (1) entities, (2) event types, (3) table relationships, (4) desired row counts, (5) noise/anomaly requirements.

---

## ⚡ Quick Reference

| Task | Pattern to use |
|---|---|
| Generate a seed entity table | `GENERATE_UUID()` + `RAND()` spine |
| Realistic names / emails | Deterministic hash from UUID |
| Timestamps with causality | `base_ts + INTERVAL CAST(...) MINUTE` chaining |
| Power-law activity | `CAST(-LOG(RAND()) * scale AS INT64)` |
| Foreign key joins | Spine table → cross/array join |
| Inject noise | Conditional `IF(RAND() < noise_rate, corrupt_val, clean_val)` |
| Multi-table consistency | Single CTE spine, all tables derive from it |
| Agentic trajectories | Step-indexed JSON arrays inside STRING columns |

---

## I. The Spine Pattern — Foundation for All Multi-Table Data

**Rule:** Every dataset starts with a single spine CTE that defines all entity IDs and their stable attributes. All other tables JOIN or derive from this spine. This guarantees entity coherence across tables.

```sql
-- ============================================================
-- SPINE: User entity table (run this first, save as a temp table
--        or use as a CTE anchor for all downstream tables)
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.users` AS

WITH spine AS (
  SELECT
    ROW_NUMBER() OVER ()                                     AS rn,
    GENERATE_UUID()                                          AS user_id,
    -- Deterministic but realistic-looking name from hash
    CONCAT(
      CASE MOD(ABS(FARM_FINGERPRINT(CAST(ROW_NUMBER() OVER() AS STRING))), 10)
        WHEN 0 THEN 'Aarav'   WHEN 1 THEN 'Priya'
        WHEN 2 THEN 'Rohan'   WHEN 3 THEN 'Sneha'
        WHEN 4 THEN 'Vikram'  WHEN 5 THEN 'Neha'
        WHEN 6 THEN 'Arjun'   WHEN 7 THEN 'Kavya'
        WHEN 8 THEN 'Siddharth' ELSE 'Ananya'
      END,
      ' ',
      CASE MOD(ABS(FARM_FINGERPRINT(CAST(ROW_NUMBER() OVER() * 7 AS STRING))), 8)
        WHEN 0 THEN 'Sharma'  WHEN 1 THEN 'Patel'
        WHEN 2 THEN 'Mehta'   WHEN 3 THEN 'Iyer'
        WHEN 4 THEN 'Nair'    WHEN 5 THEN 'Reddy'
        WHEN 6 THEN 'Joshi'   ELSE 'Singh'
      END
    )                                                        AS full_name,
    -- Realistic signup dates over 3 years (skewed toward recent)
    TIMESTAMP_SUB(
      CURRENT_TIMESTAMP(),
      INTERVAL CAST(RAND() * RAND() * 1095 * 24 * 60 AS INT64) MINUTE
    )                                                        AS signup_ts,
    -- Tier assigned by power-law: most users are 'free'
    CASE
      WHEN RAND() < 0.65 THEN 'free'
      WHEN RAND() < 0.85 THEN 'basic'
      WHEN RAND() < 0.97 THEN 'pro'
      ELSE 'enterprise'
    END                                                      AS tier,
    ROUND(18 + RAND() * 55)                                  AS age,
    CASE WHEN RAND() < 0.52 THEN 'M' ELSE 'F' END           AS gender,
    CASE MOD(ABS(FARM_FINGERPRINT(CAST(ROW_NUMBER() OVER() AS STRING))), 6)
      WHEN 0 THEN 'Mumbai'    WHEN 1 THEN 'Delhi'
      WHEN 2 THEN 'Bangalore' WHEN 3 THEN 'Hyderabad'
      WHEN 4 THEN 'Chennai'   ELSE 'Pune'
    END                                                      AS city
  FROM
    UNNEST(GENERATE_ARRAY(1, 10000)) AS _   -- Change 10000 to desired user count
)

SELECT
  user_id,
  full_name,
  LOWER(REPLACE(full_name, ' ', '.'))
    || MOD(ABS(FARM_FINGERPRINT(user_id)), 9999)
    || '@example.com'                        AS email,
  signup_ts,
  tier,
  CAST(age AS INT64)                         AS age,
  gender,
  city
FROM spine;
```

---

## II. Temporally Consistent Event Tables

**Rule:** Always derive event timestamps from the parent entity's `signup_ts`. Never generate independent random timestamps.

```sql
-- ============================================================
-- ORDERS table — temporally consistent with users.signup_ts
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.orders` AS

WITH order_spine AS (
  SELECT
    u.user_id,
    u.signup_ts,
    u.tier,
    -- Power-law order count: enterprise users order more
    CAST(
      CASE u.tier
        WHEN 'enterprise' THEN -LOG(RAND()) * 40
        WHEN 'pro'        THEN -LOG(RAND()) * 15
        WHEN 'basic'      THEN -LOG(RAND()) * 5
        ELSE                   -LOG(RAND()) * 2
      END
    AS INT64) + 1                            AS order_count
  FROM `project.dataset.users` u
),

exploded AS (
  SELECT
    user_id,
    signup_ts,
    tier,
    order_num
  FROM order_spine,
  UNNEST(GENERATE_ARRAY(1, order_count)) AS order_num
),

orders_raw AS (
  SELECT
    GENERATE_UUID()                                          AS order_id,
    user_id,
    -- Order timestamp: MUST be after signup_ts
    TIMESTAMP_ADD(
      signup_ts,
      INTERVAL CAST(RAND() *
        TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), signup_ts, MINUTE)
      AS INT64) MINUTE
    )                                                        AS order_ts,
    -- Realistic price distribution: lognormal
    ROUND(EXP(2.5 + RAND() * 2.5) * 10) / 10               AS amount_usd,
    CASE
      WHEN RAND() < 0.70 THEN 'completed'
      WHEN RAND() < 0.88 THEN 'shipped'
      WHEN RAND() < 0.95 THEN 'processing'
      ELSE 'cancelled'
    END                                                      AS status,
    tier
  FROM exploded
)

SELECT
  order_id,
  user_id,
  order_ts,
  amount_usd,
  status,
  -- Payment method correlated with tier (realism)
  CASE
    WHEN tier = 'enterprise' AND RAND() < 0.7 THEN 'invoice'
    WHEN RAND() < 0.55 THEN 'card'
    WHEN RAND() < 0.80 THEN 'upi'
    ELSE 'netbanking'
  END                                                        AS payment_method
FROM orders_raw;
```

---

## III. Reactive State Tables (Action → Reaction)

**Rule:** When an agent executes an action (e.g., refund), the downstream table MUST reflect it. Build this reactivity with deterministic CTEs, not separate inserts.

```sql
-- ============================================================
-- REFUNDS table — reacts to cancelled/returned orders
-- Refund timestamp is ALWAYS > order_ts (causal integrity)
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.refunds` AS

SELECT
  GENERATE_UUID()                                            AS refund_id,
  o.order_id,
  o.user_id,
  o.amount_usd                                               AS refund_amount_usd,
  -- Refund happens 1–72 hours AFTER the order timestamp
  TIMESTAMP_ADD(
    o.order_ts,
    INTERVAL CAST(1 + RAND() * 71 AS INT64) HOUR
  )                                                          AS refund_ts,
  CASE
    WHEN RAND() < 0.6  THEN 'customer_request'
    WHEN RAND() < 0.85 THEN 'item_not_received'
    ELSE 'defective_product'
  END                                                        AS reason,
  CASE
    WHEN RAND() < 0.90 THEN 'processed'
    ELSE 'pending'
  END                                                        AS refund_status
FROM `project.dataset.orders` o
WHERE
  o.status = 'cancelled'
  OR (o.status = 'completed' AND RAND() < 0.04)  -- 4% of completed orders also get refunded
;
```

---

## IV. Agentic Trajectory Data

**Rule:** For AI training data, each row is a full reasoning episode: observation → chain-of-thought → action → result. Store as structured JSON strings so training pipelines can parse them.

```sql
-- ============================================================
-- AGENT_TRAJECTORIES table
-- Simulates a customer-support agent resolving tickets
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.agent_trajectories` AS

WITH tickets AS (
  SELECT
    GENERATE_UUID()                                          AS trajectory_id,
    o.order_id,
    o.user_id,
    o.amount_usd,
    o.status                                                 AS order_status,
    o.order_ts,
    u.full_name,
    u.tier,
    -- Pick a realistic issue type
    CASE MOD(ABS(FARM_FINGERPRINT(o.order_id)), 5)
      WHEN 0 THEN 'refund_request'
      WHEN 1 THEN 'delivery_delay'
      WHEN 2 THEN 'wrong_item'
      WHEN 3 THEN 'payment_failure'
      ELSE 'account_query'
    END                                                      AS issue_type
  FROM `project.dataset.orders` o
  JOIN `project.dataset.users` u USING (user_id)
  WHERE RAND() < 0.15   -- 15% of orders generate a support ticket
)

SELECT
  trajectory_id,
  order_id,
  user_id,
  issue_type,

  -- OBSERVATION: what the agent sees
  TO_JSON_STRING(STRUCT(
    full_name           AS customer_name,
    tier                AS customer_tier,
    order_status        AS current_order_status,
    amount_usd          AS order_value,
    FORMAT_TIMESTAMP('%Y-%m-%dT%H:%M:%SZ', order_ts) AS order_date,
    issue_type          AS reported_issue
  ))                                                         AS observation,

  -- CHAIN-OF-THOUGHT: reasoning steps (template-varied)
  CASE issue_type
    WHEN 'refund_request' THEN
      '1. Check order status: ' || order_status ||
      '. 2. Verify order is within 30-day refund window. ' ||
      '3. Confirm payment method supports refund. ' ||
      '4. Check if prior refund exists for this order. ' ||
      '5. Tier is ' || tier || ' — apply standard SLA.'
    WHEN 'delivery_delay' THEN
      '1. Retrieve shipping carrier and tracking ID. ' ||
      '2. Check carrier API for last-mile status. ' ||
      '3. Calculate delay = today - expected_delivery. ' ||
      '4. If delay > 5 days AND tier = pro/enterprise, escalate. ' ||
      '5. Draft proactive communication to customer.'
    ELSE
      '1. Identify issue category: ' || issue_type || '. ' ||
      '2. Retrieve full order history for user. ' ||
      '3. Cross-reference with known issue patterns. ' ||
      '4. Select resolution path from policy matrix.'
  END                                                        AS chain_of_thought,

  -- ACTION: what the agent does (structured tool call)
  TO_JSON_STRING(STRUCT(
    CASE issue_type
      WHEN 'refund_request'   THEN 'initiate_refund'
      WHEN 'delivery_delay'   THEN 'query_shipping_api'
      WHEN 'wrong_item'       THEN 'create_return_label'
      WHEN 'payment_failure'  THEN 'retry_payment_gateway'
      ELSE 'lookup_account'
    END                       AS tool_name,
    TO_JSON_STRING(STRUCT(order_id AS order_id, user_id AS user_id)) AS tool_args
  ))                                                         AS action,

  -- RESULT: outcome of the action
  CASE
    WHEN RAND() < 0.88 THEN 'success'
    WHEN RAND() < 0.97 THEN 'partial_success'
    ELSE 'failure'
  END                                                        AS result_status,

  -- QUALITY LABEL: for filtering during training
  CASE
    WHEN RAND() < 0.80 THEN 'verified'
    WHEN RAND() < 0.95 THEN 'unverified'
    ELSE 'rejected'
  END                                                        AS quality_label

FROM tickets;
```

---

## V. Noise & Long-Tail Injection

**Rule:** Clean synthetic data creates brittle models. Inject realistic noise at a controlled rate (typically 2–8%) to teach agents to handle messy real-world inputs.

```sql
-- ============================================================
-- Apply noise layer on top of any clean table
-- Pattern: wrap the source table, corrupt with IF(RAND() < rate)
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.users_with_noise` AS

SELECT
  user_id,

  -- Typo injection in email (swap dot for comma ~3% of the time)
  IF(RAND() < 0.03,
    REPLACE(email, '.com', ',com'),
    email
  )                                                          AS email,

  -- Null injection for optional fields (~5%)
  IF(RAND() < 0.05, NULL, age)                              AS age,

  -- Out-of-range anomaly (~1%)
  IF(RAND() < 0.01, CAST(RAND() * 200 AS INT64), age)      AS age_raw,

  -- Duplicate-style near-match name (~2%)
  IF(RAND() < 0.02,
    UPPER(full_name),
    full_name
  )                                                          AS full_name,

  signup_ts,
  tier,
  gender,
  city

FROM `project.dataset.users`;
```

---

## VI. Diversity & Deduplication Filter

**Rule:** Before using synthetic data for training, remove near-duplicates. In BigQuery, use hashing to bucket rows and sample from each bucket.

```sql
-- ============================================================
-- Diversity filter: keep only 1 row per semantic cluster
-- Cluster = hash of (issue_type, result_status, quality_label)
-- ============================================================
CREATE OR REPLACE TABLE `project.dataset.agent_trajectories_filtered` AS

WITH ranked AS (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY
        issue_type,
        result_status,
        quality_label
      ORDER BY RAND()   -- Random sample from each cluster
    )                                                        AS rank_in_cluster
  FROM `project.dataset.agent_trajectories`
  WHERE quality_label = 'verified'
)

SELECT * EXCEPT (rank_in_cluster)
FROM ranked
WHERE rank_in_cluster <= 500;   -- Cap 500 rows per cluster for balance
```

---

## VII. Temporal Consistency Validation Queries

**Rule:** Always run these checks after generation. Fix violations before training.

```sql
-- CHECK 1: No event before user signup
SELECT COUNT(*) AS causality_violations
FROM `project.dataset.orders` o
JOIN `project.dataset.users` u USING (user_id)
WHERE o.order_ts < u.signup_ts;
-- Expected: 0

-- CHECK 2: No orphan orders (orders without a user)
SELECT COUNT(*) AS orphan_orders
FROM `project.dataset.orders` o
LEFT JOIN `project.dataset.users` u USING (user_id)
WHERE u.user_id IS NULL;
-- Expected: 0

-- CHECK 3: No refund before its order
SELECT COUNT(*) AS refund_causality_violations
FROM `project.dataset.refunds` r
JOIN `project.dataset.orders` o USING (order_id)
WHERE r.refund_ts < o.order_ts;
-- Expected: 0

-- CHECK 4: Distribution sanity — tier breakdown
SELECT tier, COUNT(*) AS cnt, ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM `project.dataset.users`
GROUP BY tier
ORDER BY cnt DESC;
-- Expected: free ~65%, basic ~20%, pro ~12%, enterprise ~3%

-- CHECK 5: Trajectory quality label distribution
SELECT quality_label, COUNT(*) AS cnt
FROM `project.dataset.agent_trajectories`
GROUP BY quality_label;
-- Aim for: verified > 75% of total
```

---

## VIII. Full Dataset Generation — Master Script Order

Run tables in this dependency order to guarantee referential integrity:

```
1. users                       (spine — no dependencies)
2. orders                      (depends on: users)
3. refunds                     (depends on: orders)
4. agent_trajectories          (depends on: orders, users)
5. users_with_noise            (depends on: users — optional layer)
6. agent_trajectories_filtered (depends on: agent_trajectories)
```

**BigQuery execution tip:** Run each as a separate `CREATE OR REPLACE TABLE` statement, not as a single multi-CTE script, to avoid hitting BigQuery's 1,000-CTE-node plan limit on large row counts.

---

## IX. Scaling Guidelines

| Target Row Count | Recommended Approach |
|---|---|
| < 100K | Single `UNNEST(GENERATE_ARRAY(1, N))` spine |
| 100K – 10M | Spine table materialized first, downstream tables as separate jobs |
| 10M – 1B | Use `TABLESAMPLE` on real data as distribution seeds + synthetic overlay |
| > 1B | BigQuery Dataform pipeline with incremental generation jobs |

---

## X. Model Collapse Prevention

When using this synthetic data to fine-tune or train models, always mix in at least **15–20% real human-generated data** or tool-execution outputs. Purely recursive synthetic training causes distribution collapse within 2–3 training cycles. Tag all rows with a `data_source` column (`synthetic_v1`, `human`, `tool_verified`) so the training pipeline can enforce mixing ratios.

```sql
-- Add provenance column to all generated tables
ALTER TABLE `project.dataset.agent_trajectories`
ADD COLUMN IF NOT EXISTS data_source STRING DEFAULT 'synthetic_v1';

ALTER TABLE `project.dataset.orders`
ADD COLUMN IF NOT EXISTS data_source STRING DEFAULT 'synthetic_v1';
```
