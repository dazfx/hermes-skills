---
name: openclaw-crash-test
description: Systematic crash-testing methodology for the OpenClaw CRM/analytics system (alef-med.com). Covers database integrity, business logic, infrastructure, and code review patterns.
version: 1.0
---

# OpenClaw Crash-Test Methodology

## System Overview
- PostgreSQL `analytics` DB on localhost:5432, user `analytics_user`
- Python venv: `/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/python3`
- Working dir: `/root/.openclaw/workspace/openclaw-mcp-servers`
- PGPASSWORD from `.env` (`POSTGRES_PASSWORD`)
- MCP servers: analytics(:8101), wazzup24(:8102), meta-ads(:8103), gateway(:18789)
- AX2 dashboard: :8080, PHP cron at `/var/www/ax/ax2_cron.php`
- Scripts: `sync_altegio.py`, `sync_amocrm.py`, `stale_leads_automation.py`, `daily_sync_with_notify.sh`

## Test Categories

### 1. Database Integrity (ALWAYS START HERE)
```sql
-- Orphan records
SELECT COUNT(*) FROM core.fact_appointment WHERE client_id IS NOT NULL AND client_id NOT IN (SELECT id FROM core.dim_client);
SELECT COUNT(*) FROM core.fact_appointment WHERE service_id IS NOT NULL AND service_id NOT IN (SELECT id FROM core.dim_service);
SELECT COUNT(*) FROM core.fact_appointment WHERE employee_id IS NOT NULL AND employee_id NOT IN (SELECT id FROM core.dim_employee);

-- NULL client_ids with revenue
SELECT attendance_status, COUNT(*), SUM(amount) FROM core.fact_appointment WHERE client_id IS NULL AND amount > 0 GROUP BY attendance_status;

-- Duplicate snapshots
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE monetary = total_paid;  -- Should DIFFER after fix

-- Unexpected statuses
SELECT attendance_status, COUNT(*), SUM(amount) FROM core.fact_appointment GROUP BY attendance_status ORDER BY count DESC;

-- Future dates
SELECT COUNT(*) FROM core.fact_appointment WHERE created_at > CURRENT_DATE;

-- Identity map conflicts
SELECT altegio_client_id, COUNT(*) FROM core.identity_map GROUP BY altegio_client_id HAVING COUNT(*) > 1;
```

### 2. Business Logic Edge Cases
- `monetary` should use `COALESCE(cash_amount, amount)`, `total_paid` uses `amount`
- `recency_days` should be `GREATEST(CURRENT_DATE - ..., 0)` — no negatives
- `first_visit_date` = `MIN(all appointments)`, not just attended
- `attendance_status = '-1'` should map to `'no_show'`
- `repeat_visits` should only include `WHERE frequency > 0`
- Segment CASE must handle NULL and negative recency_days: `WHEN recency_days IS NULL OR recency_days < 0 THEN 'prospect'`
- Check: `SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE total_paid != monetary` — should be > 0

### 3. Infrastructure Checks
- **nginx**: Check for public PHP files that shouldn't be accessible (`ax2_cron.php`, `ax2_load.php`)
- **Locks**: Use `flock`, NOT `mkdir` for cron job locks (mkdir deadlocks on SIGKILL/OOM)
- **Secrets**: No hardcoded tokens in shell scripts — use `source .env`
- **429/5xx retry**: Separate retry budgets. 429 uses `Retry-After` header, doesn't consume 5xx attempts
- **Crontab**: Check entries with `crontab -l`, verify schedules don't overlap

### 4. PHP/AX2-Specific
- `vcost()` uses `max(cost, manual_cost, first_cost)` — list price
- `vpaid()` uses `cost` only — actual paid amount
- Revenue for LTV should use `vpaid()` (cash), NOT `vcost()` (list price)
- `isRefund()` exists but isn't applied to transaction filtering — `$amt <= 0` skips refunds
- apicall() retry: separate `$retries_5xx` counter from rate-limit retries

### 5. Code Review Patterns (sync_altegio.py)
- Status mapping must include `-1: "no_show"`
- Snapshot INSERT must use `GREATEST()` for recency_days
- Snapshot `monetary` = `COALESCE(cash_amount, amount)`, `total_paid` = `amount`
- Snapshot `first_visit_date` = `MIN(created_at) FILTER (WHERE created_at <= CURRENT_DATE)` (all appts, not just attended)
- `repeat_visits` must have `WHERE frequency > 0`

### 6. Rate Limit & Retry (All APIs)
- **amo_client.py**: `_request()` has `_429_retries` param with `MAX_429_RETRIES=7` ceiling. ✓
- **wazzup24-mcp**: `api_get/api_post/api_patch` each have `_retries` param with `MAX_429_RETRIES=7` ceiling and `Retry-After`. ✓
- **ax2_cron.php / ax2_load.php**: Both have separate `$try_5xx` and `$try_429` parameters. 5xx: max 5, 429: max 10. ✓
- **sync_amo_events.py**: Has `range(3)` cap — OK. ✓
- **sync_amocrm.py**: `api_get` has separate `retries_429` counter with max 7. ✓
- **sync_altegio.py**: Both `api_get` and `api_post` have separate `retries_429` counter with max 7. ✓
- **CRITICAL**: Never use `static` for retry counters in PHP — they persist across function calls. Use function params instead.
- **CRITICAL**: Never use `while retries_5xx < N` with `continue` on 429 without a separate 429 counter — infinite loop.
- **Retry-After**: Always wrap `int()` parse in try/except — header can be HTTP-date, not just integer.

### 7. Input Validation
- `validate_client_id()` must accept numeric strings (`int("12345")`) but reject non-numeric strings (`"abc"`) with clear ValueError.
- `VALID_ATTENDANCE_STATUSES` should NOT include `'-1'` — the mapping `-1 → 'no_show'` happens at sync time in `status_map`.

### 8. DB Constraints
- `client_identity_map` MUST have `UNIQUE INDEX ON altegio_client_id WHERE altegio_client_id IS NOT NULL` to prevent duplicate identity mappings.
- Before adding: `DELETE` duplicates keeping `MAX(id)` per `altegio_client_id`.

### 9. PHP File Gotchas
- **Authorization header**: The `Bearer` line uses string concatenation with PHP constants: `'Authorization: Bearer ' . PT . ', User ' . UT`. Context masking can break this line — always verify with `php -l` after editing.
- **ax2_load.php** needs `flock(LOCK_EX|LOCK_NB)` + `register_shutdown_function` just like `ax2_cron.php`.

### 10. Bare except Anti-Pattern
- `except:` in Python catches `KeyboardInterrupt`, `SystemExit`, `GeneratorExit` — always use `except (ValueError, TypeError):` or `except Exception:`.
- Common pattern in sync scripts: `datetime.fromisoformat()` wrapped in bare `except:` → should be `except (ValueError, TypeError):`.

### 11. Hardcoded Secrets
- `sync_amo_events.py` had `DB_URL = "postgresql://analytics_user:***@localhost..."` where `***` was a LITERAL password, not a mask. Fix: read from `.env` via `ENV.get("POSTGRES_PASSWORD", "")` and construct URL at runtime.

## Verification Pattern (ALWAYS re-test after fix)
After every code fix:
1. Run the SQL query that found the bug — confirm 0 rows/0 difference
2. Check the specific file with `php -l` or Python syntax check
3. For DB fixes: `DELETE`/`UPDATE` bad data, then verify with `SELECT`

## Known Issues (already fixed, do NOT re-fix)
- SQL injection: Do NOT fix, per user rule
- `retention_monthly` cohort_sizes CTE: already fixed
- `fact_appointment.attendance_status`: use 'attended' NOT 'visited'
- Retail data: fact_retail_sale=0 is correct (no retail module)

## Cron Job
- Crash-test cronjob ID: `4bdc94c3a152`
- Model rotation: glm-5.1 / kimi-k2.6:cloud (alternates daily)
- Rotation script: `/root/.openclaw/workspace/scripts/rotate_crash_test_model.py`
- State file: `/root/.hermes/cron/crash_test_state.json`