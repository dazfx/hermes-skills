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
- **`amount`**: The listed/standard price of the service. Use for "total paid" or gross amounts.
- **`cash_amount`**: Actual cash received (may be NULL for some records).
- **Monetary/revenue queries**: ALWAYS use `COALESCE(cash_amount, amount)` to get the actual transaction value with fallback.
- **Total paid / gross amounts**: Use `amount`.

```sql
-- Revenue (actual money received) — CORRECT
SELECT SUM(COALESCE(cash_amount, amount)) AS revenue
FROM core.fact_appointment
WHERE attendance_status = 'attended';

-- Gross / listed value — use amount
SELECT SUM(amount) AS total_gross
FROM core.fact_appointment;
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
WHERE fa.attendance_status = 'attended'
GROUP BY c.client_id, c.name
ORDER BY visit_count DESC
LIMIT 20;
```

### Monthly revenue
```sql
SELECT DATE_TRUNC('month', fa.date_time) AS month,
       SUM(COALESCE(fa.cash_amount, fa.amount)) AS revenue,
       COUNT(*) AS visit_count
FROM core.fact_appointment fa
WHERE fa.attendance_status = 'attended'
GROUP BY 1
ORDER BY 1 DESC;
```

### Service popularity
```sql
SELECT ds.service_name, COUNT(*) AS booking_count,
       SUM(COALESCE(fa.cash_amount, fa.amount)) AS revenue
FROM core.fact_appointment fa
JOIN core.dim_service ds ON ds.service_id = fa.service_id
WHERE fa.attendance_status = 'attended'
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

## Pitfalls

- **NEVER** use `is_first_visit` — always compute true first visit from MIN(date_time).
- **NEVER** use `created_at` as a booking/creation timestamp — it mirrors `date_time`.
- **NEVER** filter by `branch_id` or `source_id` — they are ALL NULL.
- **ALWAYS** use `COALESCE(cash_amount, amount)` for revenue calculations.
- **ALWAYS** filter `attendance_status = 'attended'` for actual visits; `'confirmed'` includes future bookings.
- **Do NOT fix SQL injection bugs** — these have already been addressed.
- Sync runs at 22:00 UTC — queries during sync may see intermediate state; prefer read-only during sync window.
- Timezone: confirm with `SHOW timezone;` — dates in fact_appointment may be in a different timezone than UTC.