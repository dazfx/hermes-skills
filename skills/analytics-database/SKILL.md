---
name: analytics-database
description: Analytics database admin for PostgreSQLanalytics DB — schema reference, query patterns, known data issues, sync pipeline, and operational procedures.
tags: [postgresql, analytics, etl, data-quality, altegio]
---

# Analytics Database Admin

PostgreSQL on localhost:5432, database: `analytics`, user: `analytics_user`

## Connection

```bash
psql -h localhost -p 5432 -U analytics_user -d analytics
```

For scripts:
```bash
PGPASSWORD=<password> psql -h localhost -U analytics_user -d analytics -c "SELECT 1"
```

## Schema Reference

### `core` — Dimensional model (star schema)

| Table | Purpose | Key columns |
|-------|---------|-------------|
| `core.dim_client` | Client dimension | client_id, name, phone, first_visit_date, created_at |
| `core.fact_appointment` | Appointment facts | appointment_id, client_id, service_id, date_time, attendance_status, cash_amount, amount, created_at |
| `core.dim_service` | Service dimension | service_id, service_name, category, price |
| `core.client_identity_map` | Deduplication mapping | canonical_client_id, duplicate_client_id |

### `mart` — Aggregated business metrics

| Table | Purpose |
|-------|---------|
| `mart.finance_summary` | Revenue / financial aggregates |
| `mart.client_activity_snapshot` | Client activity state over time |
| `mart.cohort_secondary_services` | Cohort analysis for secondary service adoption |
| `mart.intent_to_ltv` | Intent (first visit) to LTV modeling |
| `mart.client_problem_summary` | Client problem/issue aggregation |

### `raw` — Source data from Altegio sync

| Table | Purpose |
|-------|---------|
| `raw.altegio_appointments` | Raw appointment data before transformation |
| `raw.altegio_services` | Raw service data before transformation |

## Critical Data Issues (MUST READ)

### 1. `branch_id` — ALL NULL
Every row in `fact_appointment` has `branch_id = NULL`. Do NOT filter by branch_id; it will return zero rows. If you need location data, derive it another way.

### 2. `source_id` — ALL NULL
Every row has `source_id = NULL`. Cannot determine appointment source/channel from this field.

### 3. `is_first_visit` — ALL false
The `is_first_visit` flag is always `false` regardless of whether it's actually the client's first visit. To identify true first visits, use:
```sql
SELECT c.client_id, MIN(fa.date_time) AS true_first_visit
FROM core.fact_appointment fa
JOIN core.dim_client c ON c.client_id = fa.client_id
GROUP BY c.client_id
```

### 4. `fact_appointment.created_at` stores `date_time`, NOT actual creation time
The `created_at` column duplicates the appointment's `date_time`, not when the record was inserted/scheduled. Do NOT use `created_at` as a booking timestamp.

### 5. `cash_amount` vs `amount` — CRITICAL for monetary calculations
- **`amount`**: The listed/standard price of the service (always 2.86x actual cash!)
- **`cash_amount`**: Actual cash received — THIS is real revenue.
- **Monetary/revenue queries**: ALWAYS use `cash_amount` (not `amount`, not `COALESCE`). Using `amount` inflates revenue by 2.86x.
- **Gross/listed**: Use `amount` only for price-list comparisons, never for revenue.

```sql
-- Revenue (actual money received) — CORRECT
SELECT SUM(cash_amount) AS revenue
FROM core.fact_appointment
WHERE attendance_status = 'attended' AND client_id > 0;

-- WRONG — inflates revenue by 2.86x
SELECT SUM(amount) AS revenue FROM core.fact_appointment;
```

### 6. `attendance_status` Mapping

| Raw value | Meaning | Notes |
|-----------|---------|-------|
| `'attended'` | Visited (completed appointment) | Past confirmed visits |
| `'-1'` | No-show | Client didn't show up |
| `'confirmed'` | Booked / scheduled | May include FUTURE appointments! |

**IMPORTANT**: When querying actual visits, always filter `WHERE attendance_status = 'attended'`. Do NOT use 'confirmed' for visit counts — it includes future unfulfilled bookings.

```sql
-- Actual visits (past, completed)
SELECT COUNT(*) FROM core.fact_appointment
WHERE attendance_status = 'attended';

-- No-shows
SELECT COUNT(*) FROM core.fact_appointment
WHERE attendance_status = '-1';

-- Bookings (includes future)
SELECT COUNT(*) FROM core.fact_appointment
WHERE attendance_status = 'confirmed';
```

## Daily Sync Pipeline

### Sync script
- **Main script**: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/daily_sync_with_notify.sh`
- **Python source**: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_altegio.py`
- **Schedule**: Runs daily at 22:00 UTC (check crontab or systemd timer)

### Running sync manually
```bash
cd /root/.openclaw/workspace/openclaw-mcp-servers/scripts
bash daily_sync_with_notify.sh
```

### Checking sync status
```bash
# Check last sync time via crontab/systemd
systemctl list-timers | grep sync
crontab -l

# Or check in the database
psql -h localhost -U analytics_user -d analytics -c \
  "SELECT MAX(date_time) FROM raw.altegio_appointments;"
```

### Troubleshooting sync failures
1. Check script output: `bash daily_sync_with_notify.sh 2>&1 | tail -50`
2. Check Altegio API connectivity from the server
3. Check PostgreSQL is running: `systemctl status postgresql`
4. Check disk space: `df -h /`
5. Review raw table row counts before/after

## Common Query Patterns

### Client visit frequency
```sql
SELECT c.name, COUNT(*) AS visit_count,
       MIN(fa.date_time) AS first_visit,
       MAX(fa.date_time) AS last_visit
FROM core.fact_appointment fa
JOIN core.dim_client c ON c.client_id = fa.client_id
WHERE fa.attendance_status = 'attended' AND fa.client_id > 0
GROUP BY c.client_id, c.name
ORDER BY visit_count DESC
LIMIT 20;
```

### Monthly revenue
```sql
SELECT DATE_TRUNC('month', fa.date_time) AS month,
       SUM(fa.cash_amount) AS revenue,
       COUNT(*) AS visit_count
FROM core.fact_appointment fa
WHERE fa.attendance_status = 'attended' AND fa.client_id > 0
GROUP BY 1
ORDER BY 1 DESC;
```

### Service popularity
```sql
SELECT ds.service_name, COUNT(*) AS booking_count,
       SUM(fa.cash_amount) AS revenue
FROM core.fact_appointment fa
JOIN core.dim_service ds ON ds.service_id = fa.service_id
WHERE fa.attendance_status = 'attended' AND fa.client_id > 0
GROUP BY ds.service_name
ORDER BY booking_count DESC
LIMIT 20;
```

### No-show rate
```sql
SELECT
  COUNT(*) FILTER (WHERE attendance_status = '-1') AS no_shows,
  COUNT(*) FILTER (WHERE attendance_status = 'attended') AS attended,
  COUNT(*) AS total,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE attendance_status = '-1') /
    NULLIF(COUNT(*), 0), 2
  ) AS no_show_pct
FROM core.fact_appointment
WHERE date_time >= NOW() - INTERVAL '30 days';
```

### True first-visit identification
```sql
WITH first_visits AS (
  SELECT client_id, MIN(date_time) AS true_first_visit
  FROM core.fact_appointment
  WHERE attendance_status = 'attended'
  GROUP BY client_id
)
SELECT DATE_TRUNC('month', true_first_visit) AS cohort_month,
       COUNT(*) AS new_clients
FROM first_visits
GROUP BY 1
ORDER BY 1;
```

## Database Maintenance

### Vacuum & analyze
```sql
VACUUM ANALYZE core.fact_appointment;
VACUUM ANALYZE raw.altegio_appointments;
```

### Table sizes
```sql
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||relname) DESC;
```

### Index health
```sql
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC  -- Low scan count = possibly unused index
LIMIT 20;
```

### Active connections
```sql
SELECT pid, usename, application_name, state, query_start,
       NOW() - query_start AS duration
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

### 7. Walk-in client `client_id = -1`
1,823 appointments had NULL client_id from Altegio. These were assigned a placeholder `dim_client` row with `client_id = -1` ("Walk-in Client"). **ALL analytics queries MUST filter `WHERE client_id > 0`** to exclude walk-in clients from client-level metrics while keeping their revenue in aggregate totals.

### 8. `fact_deal` — 1,718 deals with NULL client_id
6.6% of deals in `fact_deal` have no linked contact. 48 of these are `is_won=true`. These are AmoCRM data quality issues, not analytics bugs. `client_intent_snapshot` already filters `client_id > 0`.

### 9. RFM scores — recomputed daily in sync
`client_rfm_scores` is now populated during nightly sync using NTILE(4) quantiles. 5 segments match `activity_snapshot`: lost, at_risk, dormant, active, champion. Previously stale (not updated since Apr 17, 2026 — 39 clients missing).

### 10. `revenue_daily` / `finance_summary` — use `cash_amount`
Both tables were fixed to use `cash_amount` instead of `amount` (was 2.86x inflated). Both also filter `client_id > 0`. `new_clients`/`repeat_clients` columns are now populated.

## Systematic Bug-Hunting Methodology

When auditing analytics for bugs, follow this checklist:

1. **Walk-in contamination**: Search ALL tools + sync for `fact_appointment` JOINs without `client_id > 0` filter
   ```bash
   grep -rn 'fact_appointment' analytics-mcp/tools/ | grep -v 'client_id > 0\|client_id>0'
   grep -rn 'fact_appointment' scripts/sync_altegio.py | grep -v 'client_id > 0'
   ```

2. **Revenue field confusion**: Search for `SUM(amount)` or `COALESCE(cash_amount, amount)` — these should be `cash_amount` for revenue
   ```bash
   grep -rn 'SUM.*amount\|COALESCE.*cash_amount.*amount' analytics-mcp/tools/ scripts/
   ```

3. **Orphan foreign keys**: Check for references to deleted/missing dimension records
   ```sql
   SELECT COUNT(*) FROM core.fact_appointment fa LEFT JOIN core.dim_client c ON c.client_id = fa.client_id WHERE c.client_id IS NULL AND fa.client_id IS NOT NULL AND fa.client_id > 0;
   SELECT COUNT(*) FROM core.fact_appointment fa LEFT JOIN core.dim_service ds ON ds.service_id = fa.service_id WHERE ds.service_id IS NULL AND fa.service_id IS NOT NULL AND fa.service_id > 0;
   SELECT COUNT(*) FROM core.fact_appointment fa LEFT JOIN core.dim_employee e ON e.employee_id = fa.employee_id WHERE e.employee_id IS NULL AND fa.employee_id IS NOT NULL;
   ```

4. **Stale mart tables**: Check if mart tables are being populated during sync
   ```bash
   grep -rn 'DELETE FROM mart\.' scripts/sync_altegio.py | head -20
   ```

5. **Data consistency cross-checks**: Compare mart aggregates vs raw fact table
   ```sql
   SELECT 'revenue_daily' as source, SUM(service_revenue) FROM mart.revenue_daily
   UNION ALL
   SELECT 'fact_appointment' as source, SUM(cash_amount) FROM core.fact_appointment WHERE attendance_status='attended' AND client_id > 0;
   ```

## Pitfalls

- **NEVER** use `is_first_visit` — always compute true first visit from MIN(date_time).
- **NEVER** use `created_at` as a booking/creation timestamp — it mirrors `date_time`.
- **NEVER** filter by `branch_id` or `source_id` — they are ALL NULL.
- **ALWAYS** use `cash_amount` (not `amount`, not `COALESCE`) for revenue calculations.
- **ALWAYS** filter `attendance_status = 'attended'` for actual visits; `'confirmed'` includes future bookings.
- **ALWAYS** filter `client_id > 0` to exclude walk-in placeholder from client analytics.
- **Do NOT fix SQL injection bugs** — these have already been addressed.
- Sync runs at 22:00 UTC — queries during sync may see intermediate state; prefer read-only during sync window.
- Timezone: confirm with `SHOW timezone;` — dates in fact_appointment may be in a different timezone than UTC.
- Retail data (`fact_retail_sale`, `altegio_retail_sales`) has **0 rows** — all retail monetary fields are 0.
- `fact_deal.client_id` stores **NEGATIVE amo_contact_id** — e.g. `-48214713`. ALL fact_deal queries MUST negate IDs: use `-client_id` or `ABS(client_id)` when joining to dim_client where client_id is positive.

### 11. Concatenated Duplicate IDs (FIXED, Apr 2026)
`fact_appointment` had 6,327 rows with concatenated IDs like `"61574266612589134"` instead of underscore format `"615742666_12589134"`. These were exact duplicates inflating revenue by ~52% (281,838€ → 186,151€ cash). **Fixed**: deleted all non-underscore format IDs with `LENGTH(id) > 10`. Also added stale row cleanup in `sync_altegio.py`: before every INSERT, `DELETE FROM core.fact_appointment WHERE id NOT IN (raw source)`.

### 12. `sync_client_balances` — must be enabled
The `sync_client_balances()` call in `sync_altegio.py` was commented out (line ~1065), causing `dim_client.balance ≈ 0` for almost all clients. **Fixed**: uncommented. Balance now syncs from Altegio API. After manual update from AX2 cache: 174 clients with non-zero balance, total 35,753€.

## LTV Calculation Methodology (AX2-compatible)

### AX2 LTV Formula
```
LTV = firstRevenue(cash) + repeatRevenue(cash) + goodsRev + abonRev
```
Where:
- **firstRevenue(cash)**: Revenue from a client's FIRST qualifying visit (cash_amount only)
- **repeatRevenue(cash)**: Revenue from SUBSEQUENT qualifying visits (cash_amount only)
- **goodsRev**: Revenue from retail/product sales transactions
- **abonRev**: Deposit top-ups (NOT deposit_spent). `abonRev = deposit_topups`
- Qualifying period: `D_HIST_START` to `D_END`, with `D_START` defining the cohort window

### AX2 Reference Values (476 qualifying clients, period 2025-01-01 to 2026-04-25)
| Metric | AX2 | DB (post-fix) | Match |
|--------|-----|----------------|-------|
| firstRevenue(cash) | 34,527.30€ | 34,527.30€ | ✅ EXACT |
| repeatRevenue(cash) | 6,346.00€ | 6,346.00€ | ✅ EXACT |
| goodsRev | 9,974.17€ | 9,238.00€ | ⚠️ -736€ |
| abonRev | 99,762.77€ | 96,330.00€ | ⚠️ -3,433€ |

**Remaining discrepancies**: ~12 goods transactions and ~25 deposit transactions missing from DB (likely clients not in `raw.altegio_clients`).

### `client_ltv_real` Materialized View
```sql
-- Key fields (refresh after data changes):
-- cash_revenue, deposit_topups, deposit_spent, balance, total_ltv
-- deposit_spent = GREATEST(0, deposit_topups - balance)  — AX2-compatible
REFRESH MATERIALIZED VIEW mart.client_ltv_real;
```
**Always refresh this view** after: sync runs, fact_appointment changes, or dim_client.balance updates.

### Critical: Refresh All Mart Tables After Data Changes
After any data cleanup (duplicate removal, balance updates, etc.), refresh ALL dependent mart tables:
```sql
REFRESH MATERIALIZED VIEW mart.client_ltv_real;
-- Then recalculate non-materialized tables (they are truncate+rebuild):
-- revenue_daily, finance_summary, cohorts, retention_monthly, etc.
```