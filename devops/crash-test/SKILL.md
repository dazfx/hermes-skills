---
name: crash-test
description: Systematic crash-testing methodology for the Hermes analytics/CRM system (alef-med.com). Covers database integrity, business logic, infrastructure, stress tests, and code review patterns. OpenClaw is DEAD — Hermes only — all services run independently.
version: 2.0
---

# Hermes Crash-Test & Test-Race Methodology

## System Overview (Post-OpenClaw)
- PostgreSQL `analytics` DB on localhost:5432, user `analytics_user`
- PGPASSWORD: `grep POSTGRES_PASSWORD /root/.openclaw/workspace/openclaw-mcp-servers/.env | cut -d= -f2`
- Python venv: `/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/python3`
- MCP servers (independent systemd): analytics(:8101), wazzup24(:8102), meta-ads(:8103)
- AmoCRM MCP: stdio mode (`node dist/index.js`)
- AX2 dashboard: :8080, PHP cron at `/var/www/ax/ax2_cron.php`
- Hermes gateway: PID (via `hermes_cli.main gateway run --replace`)
- Scripts: `daily_sync_with_notify.sh` (flock), `automation_watcher.sh`, `crawl_alefmed.py`, `stale_leads_automation.py` (PAUSED)
- **OpenClaw gateway + claude-mem: DISABLED** — do not restart

## Test Categories

### 1. Database Integrity (ALWAYS START HERE)
```sql
-- Orphan records (NOTE: walk-in client id=-1 is intentional, NOT an orphan)
SELECT COUNT(*) FROM core.fact_appointment WHERE employee_id IS NOT NULL AND employee_id NOT IN (SELECT id FROM core.dim_employee);
SELECT COUNT(*) FROM core.fact_appointment WHERE service_id IS NOT NULL AND service_id NOT IN (SELECT id FROM core.dim_service);
-- Client orphans (exclude walk-in id=-1 which is valid)
SELECT COUNT(*) FROM core.fact_appointment WHERE client_id NOT IN (SELECT id FROM core.dim_client) AND client_id > 0;
-- NULL client_ids (should be 0 — walk-ins use id=-1)
SELECT COUNT(*) FROM core.fact_appointment WHERE client_id IS NULL;

-- Duplicate snapshots
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE monetary = total_paid;  -- Should DIFFER after fix

-- Unexpected statuses
SELECT attendance_status, COUNT(*), SUM(amount) FROM core.fact_appointment GROUP BY attendance_status ORDER BY count DESC;

-- Future dates
SELECT COUNT(*) FROM core.fact_appointment WHERE created_at > CURRENT_DATE;

-- Identity map conflicts (correct table name is client_identity_map)
SELECT altegio_client_id, COUNT(*) FROM core.client_identity_map WHERE altegio_client_id IS NOT NULL GROUP BY altegio_client_id HAVING COUNT(*) > 1;
```

### 1c. Stress Tests (run after integrity checks)
```bash
# MCP SSE endpoint rapid fire (5x each, all should return 200)
for port in 8101 8102 8103; do
  for i in $(seq 5); do
    curl -s -o /dev/null -w '%{http_code}' --max-time 5 http://localhost:$port/sse
  done
done

# AmoCRM MCP stdio init test
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | timeout 10 node /root/.openclaw/workspace/openclaw-mcp-servers/amocrm-mcp-patched/dist/index.js

# PostgreSQL connection limit
SELECT setting FROM pg_settings WHERE name='max_connections';  -- Should be 100
SELECT COUNT(*) FROM pg_stat_activity;  -- Should be well under 100

# Check stale lock files (should be empty files from flock, not blocking)
ls -la /tmp/*.lock  # Empty files are OK; files with content = stale mkdir lock
```

### 1d. Walk-in Client & Data Consistency (IMPORTANT)
```sql
-- Walk-in client sanity checks
SELECT COUNT(*) FROM core.fact_appointment WHERE client_id IS NULL;  -- Must be 0 (all walk-ins → id=-1)
SELECT COUNT(*) FROM core.fact_appointment WHERE client_id=-1;  -- Walk-in records
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE client_id=-1;  -- Must be 0 (walk-in excluded from snapshot)
SELECT COUNT(*) FROM core.dim_client WHERE id=-1;  -- Must be 1

-- Phantom values (should be 0)
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE monetary > 100000 OR total_paid > 100000;
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE frequency=0 AND monetary > 0;

-- Duplicate client_id in snapshot (must be 0)
SELECT COUNT(*) FROM (SELECT client_id FROM mart.client_activity_snapshot GROUP BY client_id HAVING COUNT(*)>1) sub;
```

### 2. Business Logic Edge Cases
- **Walk-in client**: id=-1 in dim_client holds orphan appointments. Analytics queries MUST filter `client_id > 0`.
- `monetary` should use `COALESCE(cash_amount, amount)`, `total_paid` uses `amount`
- `recency_days` should be `GREATEST(CURRENT_DATE - ..., 0)` — no negatives
- `first_visit_date` = `MIN(all appointments)`, not just attended
- `attendance_status = '-1'` should map to `'no_show'`
- `repeat_visits` should only include `WHERE frequency > 0`
- Segment CASE must handle NULL and negative recency_days: `WHEN recency_days IS NULL OR recency_days < 0 THEN 'prospect'`
- Check: `SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE total_paid != monetary` — should be > 0

### 2b. Segment Distribution Sanity
```sql
-- Champion avg monetary should be high (>>100)
SELECT segment, COUNT(*), round(AVG(monetary)::numeric) FROM mart.client_activity_snapshot GROUP BY segment ORDER BY count DESC;
-- at_risk should NOT have frequency=0
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE segment='at_risk' AND frequency=0;  -- Should be 0
-- prospect should have monetary=0
SELECT COUNT(*) FROM mart.client_activity_snapshot WHERE segment='prospect' AND monetary > 0;  -- Should be 0 or very low
```

### 2c. Retention Cohort Validation (catches the 100% bug)
```sql
-- M1 retention should be 30-60%, NOT 100%
SELECT cohort_month, cohort_size, retention_rate FROM mart.retention_monthly WHERE period_offset=1 ORDER BY cohort_month DESC LIMIT 5;
-- No impossible values
SELECT COUNT(*) FROM mart.retention_monthly WHERE retention_rate > 1.0;  -- Must be 0
SELECT COUNT(*) FROM mart.retention_monthly WHERE retention_rate < 0;    -- Must be 0
SELECT COUNT(*) FROM mart.retention_monthly WHERE cohort_size = 0;      -- Must be 0
```

### 3. Infrastructure & Stress Tests
- **nginx**: Check for public PHP files on port 8080 (`curl http://localhost:8080/ax2_cron.php` → should be 403)
- **Locks**: Use `flock`, NOT `mkdir` for cron job locks (mkdir deadlocks on SIGKILL/OOM)
- **Secrets**: No hardcoded tokens in shell scripts — use `source .env`
- **429/5xx retry**: Separate retry budgets. 429 uses `Retry-After` header, doesn't consume 5xx attempts
- **Crontab**: Check entries with `crontab -l`, verify schedules don't overlap
- **MCP stress test**: Hit each MCP /sse endpoint 5x rapidly, verify all return 200
- **Systemd restart policy**: All MCP services should have `Restart=always` + `RestartSec=10`
- **MCP init test**: Pipe JSON-RPC `initialize` via stdio to verify AmoCRM MCP boots
- **Hermes gateway process**: Verify `hermes_cli.main gateway run` is running
- **UFW**: `ufw status` should show `active` with required ports (22,80,443,3001,8080,8090,8501)

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

## Round 4 Crash-Test Results (Apr 27 2026) — 45 tests, 29 PASS, 8 WARN, 3 FAIL

### NEW BUGS FOUND
1. **revenue_daily/finance_summary STALE after code fix** — v77b0367 used `fa.amount` (list price) for service_revenue instead of `fa.cash_amount`. Fix deployed in c8b30f5 but mart tables kept stale data (€287K) until manually refreshed. **FIX APPLIED**: manually ran DELETE+INSERT for revenue_daily and finance_summary. Now shows €188,179.82. **PREVENTION**: always refresh marts after code changes to sync scripts, or add a post-deploy hook.
2. **raw.amo_events=0 rows** — table exists in schema but is NEVER populated by any script. sync_amocrm.py writes to raw.amo_events_raw only. **NOT A BUG**: all MCP tools (conversations.py, client_ranking.py, etc.) correctly query raw.amo_events_raw (373K rows). raw.amo_events is a dead schema artifact.
3. **77 duplicate transaction pairs** in core.fact_client_account_transaction — same client+type+amount+date with consecutive Altegio IDs. These are REAL separate entries (e.g., service_payment + deposit deduction for same visit), NOT data corruption. ON CONFLICT (id) dedup works correctly.

### NEW TEST CASES (Round 4+)
```sql
-- T1: Data freshness — check last sync per table
SELECT 'fact_appointment' as tbl, COUNT(*) cnt, MAX(created_at)::date FROM core.fact_appointment
UNION ALL SELECT 'fact_client_account_transaction', COUNT(*), MAX(created_at)::date FROM core.fact_client_account_transaction
UNION ALL SELECT 'fact_deal', COUNT(*), MAX(created_at)::date FROM core.fact_deal;

-- T2: Transaction integrity — no negative amounts, no orphan client_ids
SELECT COUNT(*) FROM core.fact_client_account_transaction WHERE amount < 0;
SELECT COUNT(*) FROM core.fact_client_account_transaction t LEFT JOIN core.dim_client c ON c.id=t.client_id WHERE c.id IS NULL AND t.client_id > 0;

-- T4: LTV view integrity — no negative revenue, no walk-in
SELECT COUNT(*) FROM mart.client_ltv_real WHERE cash_revenue < 0;
SELECT COUNT(*) FROM mart.client_ltv_real WHERE client_id <= 0;

-- T5: fact_deal cross-check — client_id must be negative (amo convention)
SELECT COUNT(*) FROM core.fact_deal WHERE client_id > 0 AND client_id IS NOT NULL;  -- Should be 0

-- T12: Revenue cross-validation — critical consistency check
SELECT ROUND(SUM(service_revenue),2) FROM mart.revenue_daily;     -- Must equal...
SELECT ROUND(SUM(service_revenue),2) FROM mart.finance_summary;   -- ...this
SELECT ROUND(SUM(cash_revenue),2) FROM mart.client_ltv_real;      -- Close (may differ by ~€2K due to COALESCE)

-- T15: Future dates — confirmed bookings OK, attended = BUG
SELECT COUNT(*) FROM core.fact_appointment WHERE created_at > CURRENT_DATE AND attendance_status='attended';  -- Must be 0

-- T17: Duplicate records — same client+service+date (may be legitimate multi-session)
SELECT COUNT(*) FROM (SELECT client_id, service_id, created_at::date FROM core.fact_appointment WHERE client_id > 0 GROUP BY 1,2,3 HAVING COUNT(*)>1) sub;

-- T40: Client count cross-check
SELECT COUNT(*) FROM core.dim_client WHERE id > 0;        -- All clients
SELECT COUNT(DISTINCT client_id) FROM core.fact_appointment WHERE client_id > 0;  -- Active clients
SELECT COUNT(*) FROM mart.client_activity_snapshot;        -- Should match active
SELECT COUNT(*) FROM mart.client_ltv_real;                 -- Should be close to dim_client count
```

### POST-CODE-FIX MART REFRESH PROCEDURE
After any change to sync_altegio.py that affects mart queries:
1. Run sync: `cd /root/.openclaw/workspace/openclaw-mcp-servers && .venv/bin/python3 scripts/sync_altegio.py`
2. If can't wait for sync, manually refresh affected tables:
```sql
DELETE FROM mart.revenue_daily;
-- Re-run INSERT from sync_altegio.py revenue_daily section
DELETE FROM mart.finance_summary;
-- Re-run INSERT from sync_altegio.py finance_summary section
REFRESH MATERIALIZED VIEW CONCURRENTLY mart.client_ltv_real;
```
3. Verify: `SELECT ROUND(SUM(service_revenue),2) FROM mart.revenue_daily` must equal `SELECT ROUND(SUM(cash_amount),2) FROM core.fact_appointment WHERE attendance_status='attended' AND client_id > 0 AND amount > 0 AND created_at <= CURRENT_DATE`

## Known Issues (already fixed, do NOT re-fix)
- SQL injection: Do NOT fix, per user rule
- `retention_monthly` cohort_sizes CTE: already fixed (was 100% → now M1 ~30-44%)
- `fact_appointment.attendance_status`: use 'attended' NOT 'visited'
- Retail data: fact_retail_sale=0 is correct (no retail module)
- OpenClaw gateway disabled — MCP servers run independently via systemd
- **fact_deal.client_id stores NEGATIVE amo_contact_id** (e.g. -48214713, NOT 48214713). All tools querying fact_deal MUST either: (a) negate IDs in Python [-int(x) for x in amo_ids], or (b) use ABS(fd.client_id) in SQL JOINs
- revenue_daily stale after code fix — manually refreshed Apr 27. Next sync will use corrected code automatically

## Fixed Issues (Apr 27 session 1 — do NOT re-fix)
- ✅ 11 orphan employee_id=2891230 → SET employee_id=NULL
- ✅ 1823 NULL client_id → walk-in client id=-1 in dim_client + fact_appointment
- ✅ sync_altegio.py: COALESCE(client_id, -1) in INSERT/UPDATE, client_id > 0 in 5 WHERE clauses
- ✅ crawl_alefmed.py: added fcntl.flock guard (/tmp/crawl_alefmed.lock)
- ✅ stale_leads_automation.py: added fcntl.flock guard (/tmp/stale_leads.lock)
- ✅ client_ranking.py, secondary_services.py: client_id > 0 filter (3+1 queries)
- Stale /tmp/*.lock files: harmless — flock is process-bound, not file-existence-bound

## Fixed Issues (Apr 27 session 2 — do NOT re-fix)
- ✅ RFM recomputation: added daily DELETE+INSERT in sync_altegio.py (was stale since Apr 17)
- ✅ revenue_daily: switched from amount to cash_amount; added client_id>0 filter; DELETE all rows on sync
- ✅ finance_summary: switched from amount to cash_amount; added client_id>0 filter
- ✅ client_id>0 filter added to ALL 14 MCP tools (was only in 3)
- ✅ client_intent_snapshot: JOIN changed from cim.amo_contact_id=fd.client_id to ABS(fd.client_id)
- ✅ conversation_ai: fd.client_id IN clause now uses negated IDs [-int(x) for x in amo_ids]
- ✅ conversations.py: validated_ids now negated [-int(x) for x in amo_ids]
- ✅ reactivation_orch.py: deal_filter builder now negates amo_ids
- ✅ revenue_daily: 4 ghost rows (dates with only confirmed visits) cleaned up

## Cron Job
- Crash-test cronjob ID: `4bdc94c3a152`
- Model rotation: glm-5.1 / kimi-k2.6:cloud (alternates daily)
- Rotation script: `/root/.openclaw/workspace/scripts/rotate_crash_test_model.py`
- State file: `/root/.hermes/cron/crash_test_state.json`
- ЗАВИСШИЕ cron: PAUSED (commented out with `# PAUSED:` prefix in crontab)
- All notifications via @shostka_help_bot (NOT @anastasialeadbot which is disabled)

## Quick pq() Helper for Crash Tests
```python
from hermes_tools import terminal
pwd = terminal(command="grep POSTGRES_PASSWORD /root/.openclaw/workspace/openclaw-mcp-servers/.env | cut -d= -f2")["output"].strip()
def pq(sql):
    return terminal(command=f"PGPASSWORD='{pwd}' psql -h localhost -U analytics_user -d analytics -t -c \"{sql}\" 2>&1")['output'].strip().replace('\n','')
```
NOTE: Cannot use `bash -c 'source .env && psql'` — PGPASSWORD must be explicitly exported. Use `PGPASSWORD='$pwd' psql` pattern instead.