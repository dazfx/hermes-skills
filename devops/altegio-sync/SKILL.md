---
name: altegio-sync
description: Altegio ETL sync — pull data from Altegio API into PostgreSQL raw/core/mart layers. Covers sync_altegio.py (1100+ lines), known bugs, data mappings, and troubleshooting.
version: 1.0
---

# Altegio Sync Skill

## Overview
`sync_altegio.py` is the main ETL script that pulls data from Altegio API into PostgreSQL. It's the most complex and frequently patched script in the system.

## Location
- Script: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_altegio.py`
- Venv: `/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/python3`
- Env: `/root/.openclaw/workspace/openclaw-mcp-servers/.env`
- Crontab: `0 22 * * *` (22:00 UTC = 00:00 Madrid)

## Architecture: 3-Layer ETL

### Raw Layer (raw schema)
- `raw.altegio_clients` — direct client pull from `/clients/{company_id}`
- `raw.altegio_records` — appointment records (records + clients supplement)
- `raw.altegio_services` — service catalog
- `raw.altegio_staff` — staff/employees
- `raw.altegio_transactions` — financial transactions

### Core Layer (core schema)
- `core.dim_client` — clients with first_visit_date
- `core.dim_service` — service catalog
- `core.dim_staff` — staff
- `core.fact_appointment` — appointments (deduplicated)
- `core.fact_client_account_transaction` — financial transactions

### Mart Layer (mart schema)
- 24 materialized views refreshed after sync

## Key Functions (sync_altegio.py)

| Function | Lines | Description |
|----------|-------|-------------|
| `api_get()` | 60-97 | HTTP GET with 429/5xx retry budgets |
| `api_post()` | 99-129 | HTTP POST with retries |
| `sync_services()` | 131-166 | Pull service catalog |
| `sync_staff()` | 168-199 | Pull staff list |
| `sync_clients_direct()` | 201-283 | Pull ALL clients by company_id (paginated) |
| `sync_records_and_clients()` | 285-455 | Pull records + client updates for last 730 days |
| `sync_transactions()` | 457-541 | Pull financial transactions |
| `sync_client_balances()` | 543-570 | Update dim_client.balance from Altegio |
| `run_core_transformations()` | 572-732 | Raw→Core ETL with all mappings |
| `run_mart_transformations()` | 734-1071 | Refresh 24 mart materialized views |
| `main()` | 1073-end | Orchestrate everything |

## Critical Data Mappings

### attendance_status
```
Altegio status → DB value
1 → 'attended'
2 → 'confirmed'
3 → 'cancelled'
4 → 'no_show'
-1 → 'no_show' (undocumented Altegio status)
0 → 'unknown'
```

### monetary / cash_amount
- `monetary` = `COALESCE(cash_amount, amount)` — real revenue (what client paid)
- `total_paid` = `amount` — list price
- `avg_check` uses `cash_amount`
- **IMPORTANT**: NEVER use `amount` as revenue — it includes list prices with discounts

### client_id
- `fact_deal.client_id` = NEGATIVE of `amo_contact_id`
- All queries MUST use `ABS(client_id)` or `client_id > 0` to filter walk-ins
- Walk-in client has `dim_client.id = -1` — EXCLUDE from analytics

### fact_client_account_transaction types
```
Altegio type → DB type
deposit_topup → 'deposit'
goods_transaction → 'retail'
service → 'service_payment'
```

### Dedup
- `fact_appointment` has ON CONFLICT dedup based on `altegio_record_id`
- Auto-cleanup: `DELETE FROM core.fact_appointment WHERE NOT EXISTS (SELECT 1 FROM raw.altegio_records WHERE ...)`

## Known Issues & Fixes Applied

| Issue | Fix | Date |
|-------|-----|------|
| Monetary phantom revenue | COALESCE(cash_amount, amount) | Apr 2026 |
| attendance_status=-1 unmapped | Map to 'no_show' | Apr 2026 |
| recency_days negative | GREATEST(..., 0) | Apr 2026 |
| repeat_visits phantom rows | WHERE frequency > 0 | Apr 2026 |
| 6327 concatenated dupes | DELETE duplicate records | Apr 2026 |
| fact_client_account_transaction missing | Added INSERT...ON CONFLICT ETL | Apr 2026 |
| sync_client_balances commented out | Uncommented | Apr 2026 |
| 429 retry infinite loop | Separate 429/5xx budgets, Retry-After | Apr 2026 |

## Running

```bash
# Full sync
cd /root/.openclaw/workspace/openclaw-mcp-servers
.venv/bin/python3 scripts/sync_altegio.py

# Check logs
tail -f /var/log/analytics_sync.log
```

## Troubleshooting

### Sync failures
1. Check API tokens in `.env` (ALTEGIO_PARTNER_TOKEN, ALTEGIO_USER_TOKEN)
2. Check PostgreSQL: `systemctl status postgresql`
3. Check 429 rate limits in logs
4. Run manually to see errors

### Data quality checks
```sql
-- Check for duplicates
SELECT altegio_record_id, COUNT(*) FROM core.fact_appointment GROUP BY 1 HAVING COUNT(*) > 1;

-- Check revenue sanity
SELECT SUM(cash_amount), SUM(amount) FROM core.fact_appointment;

-- Check mart freshness
SELECT relname, last_autovacuum FROM pg_class WHERE relnamespace = 'mart'::regnamespace;
```

## Pitfalls
- NEVER use `amount` column as revenue — always `cash_amount` or `COALESCE(cash_amount, amount)`
- Walk-in client (id=-1) must be excluded with `client_id > 0`
- `attendance_status` has undocumented -1 value
- Transactions only load for clients in `raw.altegio_clients` — clients outside this set are missed
- `first_visit_date` backfill requires `MIN(fact_appointment.created_at)` per client