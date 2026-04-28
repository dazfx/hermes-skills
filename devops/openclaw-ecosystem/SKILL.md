---
name: openclaw-ecosystem
description: Legacy system reference — OpenClaw gateway DISABLED (Apr 27). All functionality migrated to Hermes Agent. MCP servers run as independent systemd services. Use hermes-ecosystem instead.
version: 3.0
deprecated: true
---

# OpenClaw Ecosystem — Audit & Maintenance

## Migration Status (Apr 27 2026) — COMPLETE ✅

OpenClaw gateway and claude-mem-worker are **STOPPED and DISABLED**. Hermes operates fully independently.

| What | Hermes Replacement | Status |
|------|-------------------|--------|
| Telegram bot (@anastasialeadbot) | @shostka_help_bot (Hermes bot) | ✅ Active (21 commands) |
| MCP servers (analytics, meta-ads, wazzup24) | Same systemd services | ✅ Independent |
| amocrm MCP | In Hermes config (patched version, stdio) | ✅ Active |
| context7 MCP | In Hermes config (npx) | ✅ Active |
| tolaria MCP (Obsidian) | In Hermes config (node) | ✅ Active |
| claude-mem (vector memory) | Hermes MEMORY.md + session_search | ✅ Replaced |
| openclaw-web-search | Hermes web toolset | ✅ Replaced |
| Cron tasks (8 jobs) | System crontab | ✅ Independent |
| Streamlit dashboard (:8501) | Standalone systemd | ✅ Running |
| AX2 PHP dashboard (:8080) | Standalone | ✅ Running |
| FileBrowser (:8090) | Systemd + nginx proxy | ✅ Running |

### If re-enabling is ever needed:
```bash
systemctl enable openclaw-gateway && systemctl start openclaw-gateway
systemctl enable claude-mem-worker && systemctl start claude-mem-worker
```

### Service Migration Audit Methodology (reusable pattern):
1. **Map dependencies:** grep all scripts/configs for references to the service
2. **Identify independent components:** MCP servers run as separate systemd services, not spawned by gateway
3. **Update monitoring scripts:** Remove the service from health-check arrays in automation_watcher.sh, daily_report.sh, task_watchdog.sh
4. **Update restart scripts:** Rewrite restart_mcp_stack.sh to skip gateway
5. **Verify file-based deps:** .env files, node_modules, dist/ bundles remain on disk — not served by gateway
6. **Test each layer:** MCP SSE endpoints, PostgreSQL, Telegram bot API, AmoCRM stdio
7. **Stop services, verify again, then disable**
8. **Archive data:** Copy daily memory files to ~/.hermes/archive/

**Key insight:** The `/root/.openclaw/workspace/` directory MUST NOT be deleted — MCP servers still run from it.

---

## Architecture

| Component | Port | Type | Tools |
|-----------|------|------|-------|
| ~~openclaw-gateway~~ | ~~18789~~ | ~~STOPPED~~ | — |
| Analytics MCP | 8101 (SSE) | FastMCP + uvicorn | 70 (69 OK + 1 fixed) |
| wazzup24-mcp | 8102 (SSE) | FastMCP + uvicorn | 12 (all functional, 3 stubs) |
| meta-ads-mcp | 8103 (SSE) | FastMCP | 31 |
| amocrm-mcp | stdio (node) | Node (local patched fork at amocrm-mcp-patched/) | 19 |
| context7 | stdio (npx) | Node | — |
| Altegio (ETL) | — | sync_altegio.py script | — |
| PostgreSQL | 5432 DB=analytics | asyncpg | 64 tables |
| Streamlit | 8501 | Dashboard | — |

**Data volumes:** 23,973 clients (16,084 amoCRM + 7,889 Altegio), 19,256 appointments, 25,702 deals, 24,001 conversation threads, 6,899 won deals, 359 revenue days.
**Altegio = NOT a separate MCP.** It's an ETL sync script (`scripts/sync_altegio.py`) pulling data from Altegio API into PostgreSQL (raw_altegio_* → core → mart). ALTEGIO_COMPANY_ID=1285692.
**amoCRM subdomain:** nkataeva07gmailcom

**Workspace:** openclaw-mcp-servers/ under OpenClaw workspace dir
**Config:** openclaw.json (gateway), Hermes config.yaml
**Hermes MCPs:** meta-ads (8103), analytics (8101), wazzup24 (8102), context7 (npx), amocrm (node, local patched path).
**⚠️ Hermes must be RESTARTED after editing config.yaml** for new MCP servers to appear as `mcp_*` tools. Cannot self-restart — user must do it.

## Crash-Test Methodology

1. **Check running status:** `systemctl status <service>`, check listening ports (`ss -tlnp`), check processes (`ps aux | grep mcp`)
2. **List tools:** Count `@mcp.tool` decorators in server.py; get SSE session for live tool listing
3. **Direct DB testing:** Use psql against core/mart/raw schemas. Load credentials from the same .env the service uses.
4. **Source code audit patterns:**
   - `grep -n "json.dumps" tools.py api.py` — double serialization check
   - `grep -n "@validator\|@root_validator\|\.dict()" models.py` — Pydantic v2 migration check
   - `grep -n "paginate" api.py` — pagination check
   - `grep -n "WHERE.*{" tools/*.py` — SQL injection check (f-strings in SQL)
   - `grep -n "raise_for_status\|401\|190\|token.*expired" api.py` — error handling check
5. **API endpoint testing:** Use Python urllib/httpx to call external APIs (Meta, Wazzup24) directly. **CRITICAL:** Meta API account IDs need `act_` prefix for API calls (e.g., `act_1046675009730979`)
6. **Data quality checks:** NULL client_ids, future-dated rows, empty mart tables, stale sync timestamps
7. **Validate function testing:** Test `validate()` with both string class names AND direct class references — previous session found string args don't work
8. **Token detection testing:** Create a fresh `MetaAdsAPI()` client after modifying env vars — runtime attribute changes don't propagate to headers
9. **SSE MCP protocol testing:** GET `/sse` for session_id, then POST to `/messages/?session_id=XXX` with JSON-RPC body
10. **Automated crash-testing:** Use `delegate_task` to spawn 3 parallel sub-agents testing each MCP server (Meta Ads, Analytics, Wazzup24+amoCRM). Each sub-agent calls every tool with valid and edge-case params, checks response structure, and reports bugs.
11. **INTERVAL SQL injection pattern:** PostgreSQL INTERVAL literals like `INTERVAL '{months_after} months'` can't use `$N` parameterization. **Fix:** `max(int(param), 1)` to enforce positive integer. Same pattern for days.
12. **_validate_id() helper pattern:** For MCP tools accepting ID parameters, add `if not _validate_id(param): return {"error": "..."}` at function start. `_validate_id` checks `isinstance(x, int) and x > 0`.

## Verification Results (2026-04-20 deep audit + 2026-04-21 re-test)

### Fix 1: Double serialization — ✅ VERIFIED
- tools.py: 0 `json.dumps` remaining. `object_story_spec` passes as dict directly.
- api.py: `targeting` uses `targeting if isinstance(dict) else json.loads(targeting) if isinstance(str)`.
- api.py: `time_range` uses `json.dumps({...})` (correct: params dict, not json= body).

### Fix 2: `__all__` completeness — ✅ VERIFIED
- 31 `async def` functions = 31 entries in `__all__` (lines 510-542). Exact match.

### Fix 3: time_range f-string → json.dumps — ✅ VERIFIED
- Lines 144, 157 in api.py: both use `json.dumps({"since":..., "until":...})`.

### Fix 4: Pydantic v2 — ✅ VERIFIED (with caveats)
- `@validator` / `@root_validator`: 0 occurrences ✅
- `@field_validator`: 3 occurrences, with `@classmethod` ✅
- `@model_validator(mode='after')`: 2 occurrences ✅
- `.dict()`: 0 occurrences ✅, `.model_dump()`: 1 occurrence ✅
- `CreateCampaignParams.daily_budget: Optional[PositiveInt] = None` ✅
- **⚠️ NEW BUG: `validate()` function** takes `model_class` as class arg, but called with string like `validate("CreateCampaignParams", ...)` → `'str' object is not callable`. Need string→class mapping or direct class import.
- **FIXED (Round 2):** `CreateAdSetParams.daily_budget` was `PositiveInt` (required) at models.py:148 but api.py used `Optional[int]` — now synchronized as Optional in all 3 files.

### Fix 5: Pagination — ✅ VERIFIED
- `get_campaigns`, `get_ad_sets`, `get_ads`: all pass `paginate=True`.
- Live test: returns 47 accounts, 353 campaigns (full pagination working).

### Fix 6: Token expiry detection — ✅ VERIFIED (with caveat)
- Line 85: `if 401 or error_code in (190, 102) or error_subcode in (463, 467, 458, 460)`.
- Live test with `EAABAAAAAAAAAAINVALID` → `"TOKEN EXPIRED or INVALID (code=190)"` ✅
- **⚠️ CAVEAT:** `MetaAdsAPI.__init__` reads token at init and sets `self.headers["Authorization"]`. Changing `api.access_token` attribute later does NOT update headers — must recreate client or update headers manually.
- **⚠️ CAVEAT:** Some "plausible-looking" bad tokens get empty `data: []` with 200 OK from Meta instead of error. Should add empty-data detection in `_make_request`.

### Fix 7: Wazzup24 — ✅ FIXED (Round 2)
- `/v3/templates` → uses `/v3/channels` instead (templates managed in Meta Business Suite)
- `/v3/message` — correct path (was `/v3/messages/send`)
- `/v3/messages/{id}/status` → removed, tool returns hint to use webhooks
- False "sent" status → fixed: both send tools check `"error" in result` before returning "sent"
- `send_waba_template_to_segment` — still a stub/preview (by design, requires dry_run=false)
- `get_waba_campaign_report` — still "not_implemented_yet"
- Variable regex: `{{1}}` and `{var}` both supported now

### Fix 8: Analytics SQL injection — ✅ FIXED (Round 2)
- `db.py` stoplist: `sql_escape()` function escapes `\`, `'`, `%`, `_`; `int()` validation for IDs; `ESCAPE '\\'` clause in LIKE
- `cohorts_advanced.py` `_diag_exclude_sql`: parameterized LIKE with ESCAPE clause
- `conversation_ai.py`: already uses `$1, $2` parameterized queries (verified)
- `messaging.py:61`: `int(days_back)` — safe (integer casting)

### Fix 9: CreateAdSetParams.daily_budget — ✅ FIXED (Round 2)
- models.py: `Optional[PositiveInt] = Field(None)` ✅ (was already correct)
- api.py: `daily_budget: Optional[int] = None`, conditionally added to data dict ✅
- tools.py: `daily_budget: Optional[int] = None`, parameter moved after required args ✅

## Known Bugs (as of 2026-04-21)

### Analytics-mcp — FIXED (2026-04-21)
- ✅ `personalize_message_from_conversation_tool` — `NameError: 'value_after'` because PostgreSQL JSON path `#>> '{value_after,...}'` was inside Python f-string → Python interpreted `{value_after,...}` as format expression. **Fix:** escape to `#>> '{{value_after,...}}'` in `reactivation_orch.py:123`. **Pattern to check:** any `#>> '{...}'` inside f-string queries must use doubled braces `#>> '{{...}}'`.
- ✅ `validate_client_id()` in `validation.py` (TWO copies: root + tools/) rejected negative client_ids (`client_id <= 0`), but amoCRM IDs are negative in dim_client. **Fix:** changed to `client_id is None or client_id == 0`. All tools using `validate_client_id()` now accept negative amoCRM IDs like `-44766261`.
- ✅ `segmentation.py` ILIKE injection — `service_name` passed to `f"%{service_name}%"` for ILIKE without escaping `%` or `_`. **Fix:** escape wildcards + `validate_string_not_empty()`.
- ✅ `segmentation.py` `segment` param — not validated via `validate_enum(VALID_SEGMENTS)`. **Fix:** added validation.
- ✅ `conversations.py` + `identity.py` `ids_str` — `",".join(str(x) for x in ids)` without int validation could allow string injection. **Fix:** `[int(x) for x in ids]` before join.
- ✅ `wazzup24-mcp/server.py` template double-replace bug — L95-96 did `.replace({{k}})` then `.replace({k})` in same loop, causing already-replaced values to be clobbered. **Fix:** split into two passes ({{N}} first, then {var}).
- ✅ `cohorts_advanced.py` SQL injection via `min_amount` — passed into f-string SQL without parameterization. **Fix:** parameterized with `$N` placeholder and `float()` cast.
- ✅ `clients.py` RFM `order` param SQL injection — passed directly into ORDER BY. **Fix:** validated to only accept `asc`/`desc`.
- ✅ `segmentation.py` `min_visits` injection — passed into SQL without validation. **Fix:** `max(int(min_visits), 1)`.
- ✅ INTERVAL injection in 6 analytics files — `months_after`, `days_active`, `days_back`, `days_lost`, `days_window`, `days` params passed into SQL INTERVAL literals without int validation. **Fix:** wrapped in `max(int(param), 1)` or `int()`. Cannot use $N parameterization for INTERVAL literals in PostgreSQL.

### Meta-ads-mcp — REMAINING (from crash-test #3)
- `validate("CreateCampaignParams", ...)` — passes string, function expects class → TypeError (needs MODEL_REGISTRY fix)
- Token detection doesn't catch tokens that return empty data with 200 OK
- `level` param in `get_insights` has no enum validation — invalid levels silently return empty
- `monitor.py` rate limiter race condition — `lastRequestTime` read/write not atomic under concurrency
- `429` rate-limit responses consume retry budget (3 max), should not count against retries
- ✅ FIXED (2026-04-21): `message` field removed from Meta API v21.0 — was crashing `get_ads`, `get_ad_details`, `get_creative`, `get_campaign_details`. Removed from api.py lines 135, 176 and tools.py line 627.
- ✅ FIXED (2026-04-21): `optimization_goal` removed from Meta API v21.0 for Campaign resource — was crashing `get_campaign_details`. Removed from tools.py line 609.
- ✅ FIXED (2026-04-21): `get_insights` applied `act_` prefix to campaign/ad IDs → 403 error. Now conditionally normalizes: `act_` prefix only for `level="account"`, raw `object_id` for campaign/adset/ad levels.
- ✅ FIXED (2026-04-21): `_validate_id()` helper added to tools.py — validates positive int for ID params. Applied to 20 functions: get_insights, get_creative, update_campaign, update_ad_set, update_ad, pause_campaign, activate_campaign, duplicate_campaign, monitor_campaign_health, monitor_auto_optimize, get_campaign_details, get_ad_details, update_ad_creative, pause_ad_set, activate_ad_set, pause_ad, activate_ad, create_lead_ad, create_instagram_ad, get_campaign_insights.
- ✅ FIXED (2026-04-21, crash-test #3): `daily_budget` stringified with `str()` in `create_campaign`, `update_campaign`, `update_ad_set` (was int, Meta API requires string)
- ✅ FIXED (2026-04-21, crash-test #3): `duplicate_campaign` reads `copied_campaign_id` instead of `id` (correct Meta API response key)
- ✅ FIXED (2026-04-21, crash-test #3): `monitor.py` — `insights[0]` → `insights` (get_account_insights returns dict, not list)
- ✅ FIXED (2026-04-21, crash-test #3): `get_creative` — handles `{data: [...]}` array response (was expecting single dict)
- ✅ FIXED (2026-04-21, crash-test #3): `create_ad` — requires `creative_id`, returns error if None (was silently creating unusable ad)
- ✅ FIXED (2026-04-21, crash-test #3): `_validate_id` added to `get_all_ads_for_campaign`
- ✅ FIXED (2026-04-21, crash-test #3): `create_ad_creative` docstring — `message` param marked as deprecated (v21.0)
- ✅ FIXED (2026-04-21, crash-test #3): `level` enum validation in `get_insights` (tools + api layer)
- ✅ FIXED (2026-04-21, crash-test #3): `create_ad_set` targeting dict mutation — creates copy `{**targeting, ...}` instead of in-place mutation
- ✅ FIXED (2026-04-21, crash-test #3): Pagination safety — `prev_url` check prevents infinite loop on duplicate `next` URL
- ✅ FIXED (2026-04-21, crash-test #3): 429 rate limit — clear error message with `RATE LIMITED (429)` prefix and `Retry-After` header
- ✅ FIXED (2026-04-21, crash-test #3): JSON decode safety in error handler — `response.json()` wrapped in try/catch
- REMAINING: `monitor.py` rate limiter race condition — `lastRequestTime` read/write not atomic under concurrency

### Streamlit Dashboard — FIXED (2026-04-22, crash-test #4 2026-04-23)

**Files:** `/root/.openclaw/workspace/dashboard/app.py`, `queries.py`, `meta_ads.py`, `config.py`

- ✅ FIXED: SQL INTERVAL injection — `days`/`months` params `max(int(param), 1)`
- ✅ FIXED: LIMIT injection — `max(min(int(limit), 500), 1)` to clamp 1-500
- ✅ FIXED: ILIKE injection — escape + `ESCAPE '\\\\'` clause
- ✅ FIXED: Missing try/except — `fetch_df()`/`fetch_one()` with `st.error()` fallback
- ✅ FIXED: NULL description crash — `cur.description is None` check
- ✅ FIXED: NaN display — `.dropna().mean()`
- ✅ FIXED: Divide by zero — `repeat_pct` safe default
- ✅ FIXED: IndexError — `ACCOUNTS[0] if ACCOUNTS else None`
- ✅ FIXED (crash-test #4): Cohort LTV `extract(month from age())` truncated months to 0-11 → broke cohorts >1yr. **Fix:** `(extract(year from age)*12 + extract(month from age))`
- ✅ FIXED (crash-test #4): `st.error()` in queries.py without `import streamlit as st` → NameError on DB errors. **Fix:** added import
- ✅ FIXED (crash-test #4): Sidebar `last_sync` vs actual column `last_altegio_sync` — sync time never shown. **Fix:** show both Altegio + amoCRM sync times
- ✅ FIXED (crash-test #4): KPI "В зоне риска (60-90д)" — SQL actually counts 30-90d. **Fix:** label → "30-90д"
- ✅ FIXED (crash-test #4): `deposit_revenue` column name mismatch (SQL returns `deposit_service_revenue`) → formatting lost. **Fix:** correct column name
- ✅ FIXED (crash-test #4): `cancel` missing from exclusion keywords in hardcoded SQL. **Fix:** added
- ✅ FIXED (crash-test #4): Empty campaign status filter shows ALL incl. DELETED. **Fix:** default ACTIVE+PAUSED
- ✅ FIXED (crash-test #4): Unused variable `cid` in meta_ads.py. **Fix:** removed

**Remaining (low priority):**
- `load_ads_summary(d_from, d_to)` ignores date params — always "yesterday"
- Hardcoded password in `app.py:22` and DB password in `config.py:1`
- `_sync_get()` in `meta_ads.py` has no error handling for non-200 status codes
- `account_id` in Meta Ads functions not validated for path traversal

### AX2 PHP Dashboard — FIXED (2026-04-22, crash-test #4 2026-04-23)

**Files:** `/var/www/ax/index.php`, `ax2_cron.php`, `ax2_load.php`

- ✅ FIXED: ISO week bug — `date('Y-W')` → `date('o-W')` in both files
- ✅ FIXED: PHP bool coercion — `(bool)"0"` → `=== '1'`
- ✅ FIXED: XSS — `htmlspecialchars()` on `$inputDStart`/`$inputDEnd`
- ✅ FIXED: mb_strtolower missing — `apt-get install php8.3-mbstring`
- ✅ FIXED (crash-test #4): `jsonW()` non-atomic — write goes directly to cache.json, interruption = corrupted JSON → dashboard crash. **Fix:** write to temp file + `rename()` (atomic on same filesystem)
- ✅ FIXED (crash-test #4): `index.php` crashes on corrupted cache.json: `null['meta']` → Warning. **Fix:** `is_array($data)` check before accessing

**Additional fixes (2026-04-24):**
- ✅ FIXED: `ax2_cron.php` — PHP flock concurrency guard (`LOCK_EX | LOCK_NB` + `register_shutdown_function`) prevents parallel runs
- ✅ FIXED: `firstRevenue`/`repeatRevenue` used `vcost()` (max of cost/manual_cost/first_cost = list price) instead of `vpaid()` (actual cost paid). **Phantom revenue ~€60K.** Fix: both now use `$av['cash']` (vpaid). LTV formula consolidated: `$firstRevenue + $repeatRevenue + $goodsRev + $abonRev`
- ✅ FIXED: nginx deny for `ax2_cron.php` — direct HTTP access blocked (`location ~ ^/ax2_cron.php$ { deny all; }`)
- ✅ FIXED: crontab changed from `curl localhost:8080/ax2_cron.php` to `php -f /var/www/ax/ax2_cron.php` (CLI, not web)
- ✅ FIXED: ax2_cron schedule moved from 22:05 UTC to 23:30 UTC (avoids overlap with daily_sync)
- ✅ FIXED: ax2_cron crontab now has `timeout 7200` (2hr max)

**Remaining (low priority, not fixed):**
- Hardcoded API credentials in source (`PT`, `UT` constants in ax2_load.php and ax2_cron.php).
- No file locking on `cache.json`/`progress.json` — race condition if two requests hit simultaneously.
- No CSRF protection on destructive `action=reset` endpoint.
- `curl` handles never closed (memory leak in long-running process).

### Wazzup24-mcp — FIXED (crash-test #3)
- ✅ FIXED: `send_waba_template_to_segment` / `get_waba_delivery_status` / `get_waba_campaign_report` — marked `[STUB — NOT IMPLEMENTED]`
- ✅ FIXED: `list_waba_templates` → `list_waba_channels` (function name + docstring)
- ✅ FIXED: Template double-replace — single `re.sub` pass prevents variable values from being re-replaced
- ✅ FIXED: `get_chat_window_link` — returns explicit error when `scope='card'` without required filter params
- ✅ FIXED: Connection pooling — shared `httpx.AsyncClient` via `_get_client()` instead of per-request creation
- REMAINING: `send_waba_template_to_segment` still a stub (returns "preview" only)
- REMAINING: `get_waba_delivery_status` still a stub (returns "not_implemented")
- REMAINING: `get_waba_campaign_report` still a stub (returns "not_implemented")
- ✅ FIXED (2026-04-22): `preview_waba_template` — XSS escaping via `html.escape()` on template_name and variable values
- ✅ FIXED (2026-04-22, crash-test #3 final): `set_webhooks` — returns real API error message instead of "configured" on HTTP 400
- ✅ FIXED (2026-04-22, crash-test #3 final): `send_whatsapp_message` — validates empty phone number (after stripping non-digits) and empty channel_id before API call
- ✅ FIXED (2026-04-21): `send_waba_template_to_client` — uses `chatType`+`chatId` format, validates channelId type (UUID vs type string)
- ✅ FIXED (2026-04-21): `get_unanswered_counter` without user_id — now calls `/unanswered` (all users) instead of hardcoded user 12364798
- ✅ FIXED (2026-04-21): `validate_waba_template_variables` — added `template_body` param so it searches template content, not just the name
- ✅ FIXED (2026-04-21): `api_post`/`api_patch` — removed unreachable `raise_for_status()` after early return
- ✅ FIXED (2026-04-21): `set_webhooks` now extracts actual API error from `body.message`/`body.error` instead of returning generic "HTTP 400"
- ✅ FIXED (2026-04-21): `send_whatsapp_message` validates empty phone before API call
- ✅ FIXED (2026-04-21): `send_waba_template_to_client` sent `phone` key instead of Wazzup24 v3 API format (`chatType`+`chatId`) → 400 INVALID_MESSAGE_DATA. Now uses `chatType: "whatsapp"` and `chatId` with cleaned phone number.

### amoCRM MCP — FIXED (crash-test #3)
- ✅ FIXED: `.catch(() => ({}))` → `.catch((err) => { console.error(...); return {}; })` — logs auth/server errors
- ✅ FIXED: `total` field now uses `_total_items` from API response when available
- ✅ FIXED: All `if (params.X)` → `if (params.X != null)` for numeric params (page, limit, entity_id, responsible_user_id) across 6 files
- ✅ FIXED: `response.json()` wrapped in try/catch — returns `{}` on empty body
- ✅ FIXED: `parseInt` on `Retry-After` — handles HTTP-date format (NaN → 2000ms default)
- ✅ FIXED: `create_contact` requires at least one of name/first_name/last_name
- ✅ FIXED: `entity_id` requires `entity_type` in tasks — returns error if missing
- ✅ FIXED: `process.env.AMOCRM_ACCESS_TOKEN=***` broken JS in token refresh → replaced with `data.access_token`/`data.refresh_token`
- REMAINING: Rate limiter race condition under concurrent requests
- REMAINING: 429 responses consume retry budget (should not count)

✅ FIXED (2026-04-21): `search` queries `/leads`, `/contacts`, `/companies` in parallel (in local patched fork)
✅ FIXED (2026-04-21): `list_*` tools have Zod `.min(1).int()` on limit, `if (params.X != null)` for numeric checks
- **amoCRM local patched fork** — at `/root/.openclaw/workspace/openclaw-mcp-servers/amocrm-mcp-patched/`. Original npm package `@theyahia/amocrm-mcp` has unfixed bugs (404 search, no limit validation). Always use local patched path in Hermes config: `command: node`, `args: [/path/to/amocrm-mcp-patched/dist/index.js]`. Do NOT revert to `npx @theyahia/amocrm-mcp`.
- ✅ 81 async functions across 18 analytics-mcp files now have `try/except Exception` wrappers

### Daily Sync Bugs — FIXED (2026-04-21, commit abe00b5+)
- ✅ **sync_altegio.py birth_date crash** — asyncpg got string `'1991-09-01'` instead of `date` object because `date.fromisoformat()` can't parse timezone-aware strings like `1991-09-01T00:00:00+02:00`. **Fix:** 3-level parser: `date.fromisoformat()` → `datetime.fromisoformat().date()` → `None`. Also handles `datetime` objects from API.
- ✅ **sync_amocrm.py status_id→stage_id** — INSERT used `status_id` but `core.fact_deal` column is `stage_id`. Crashed every day. **Fix:** renamed column in INSERT + ON CONFLICT.
- ✅ **sync_amocrm.py deal_id→lead_id** — `fact_conversation_thread` has `lead_id`, not `deal_id`. **Fix:** renamed column + added `EXISTS` check + `-contact_id` for dim_client negative ID convention.
- ✅ **sync_amocrm.py 5,987 contacts without phone excluded from dim_client** — `WHERE ac.phone IS NOT NULL` skipped 37% of amo contacts, causing FK violations on `fact_conversation_thread`. **Fix:** removed phone filter, kept phone-based dedup only. Added `COALESCE(EXCLUDED.phone, dim_client.phone)` for safe updates.
- ✅ **sync_amocrm.py stage_name always empty** — `raw.amo_leads.status_name` hardcoded as `""`, never populated. **Fix:** LEFT JOIN with `core.dim_stage` → `COALESCE(ds.name, al.status_name, '')`. Now 100% of deals have stage_name.
- ✅ **sync_amocrm.py is_won/is_lost/closed_at/source_id missing** — fact_deal had NULL for these analytics-critical columns. **Fix:** `is_won = (status_id = 142)`, `is_lost = (status_id = 143)`, `closed_at` from `updated_at` for closed deals, `source_id` from raw data.

### Analytics-mcp — DATA QUALITY REMAINING
- 1,795 fact_appointment + 1,671 fact_deal rows with NULL client_id
- 626 future-dated rows in fact_appointment (normal: Altegio bookings for next 30 days) — **ETL now excludes them from historical analytics**
- **FIXED (2026-04-23):** 29 phantom future finance_summary rows → added `created_at <= CURRENT_DATE` + `HAVING > 0` filters
- **FIXED (2026-04-23):** Negative recency_days in client_activity_snapshot → COALESCE fallback now filters `created_at <= CURRENT_DATE`
- **FIXED (2026-04-23):** 19,309 blanket `inactive` → replaced with `prospect` (booked never visited), `lead` (amoCRM with conversations), `unclassified` (no data at all)
- **FIXED (2026-04-23):** retention_monthly always showed 100% → `cohort_size` was PER PERIOD, not fixed at cohort entry. Now uses `cohort_sizes` CTE joined to `period_activity`. After fix: Apr 2025 M1=58.4%, M2=39.2%, M3=29.3%
- CAC = 0 everywhere (no marketing cost data)
- intent_to_ltv, client_problem_summary, cohort_secondary_services: EMPTY tables
- message_log: only 14 rows
- `branch_id` ALL NULL in fact_appointment (19,256 rows) — branch filters return empty
- `source_id` ALL NULL in fact_appointment — dimension grouping produces meaningless all-NULL aggregates
- `is_first_visit` ALL false in fact_appointment
- ~1,893 fact_appointment rows have `service_id IS NULL` — JOINs with dim_service silently drop ~10%

**DEEP AUDIT 2026-04-23 (3 problems investigated):**

**① Segment 80% "inactive" — BUG (not just "business abroad")**
- 19,309 out of 24,005 dim_client = 80.4% inactive. Root cause: `client_activity_snapshot` only has 2,273 clients (those with fact_appointment). The rest get `segment='inactive'` as default in `daily_sync_with_notify.sh` lines 122-125: `UPDATE core.dim_client SET segment = 'inactive' WHERE segment IS NULL OR segment = ''`
- 13,690 amoCRM contacts (id<0) have NO identity_map link to Altegio → automatically "inactive"
- Only 2,423 of 16,113 amoCRM contacts have identity_map link (15%)
- **BUG in recency_days formula** (sync_altegio.py line ~722): `COALESCE(MAX(created_at) FILTER (WHERE attended), MIN(created_at))` — if no attended visits, falls back to `MIN(created_at)` which can be a FUTURE date → **negative recency_days** → `recency_days <= 30` → falsely assigned `active` segment
- 19 clients have negative recency_days, 28 "active" clients have 0 completed appointments (never actually visited)
- **FIX NEEDED:** (a) Add `WHERE created_at <= CURRENT_DATE` to COALESCE fallback, (b) Add segments like `lead`/`prospect`/`unlinked` instead of blanket `inactive`, (c) Exclude zero-visit clients from snapshot or set segment only when completed_appointments > 0

**② 626 future appointments — NORMAL for CRM, but POLLUTES analytics**
- Altegio API intentionally pulls `end_date = now + 30 days` (sync_altegio.py line 266) — correct behavior
- `core.fact_appointment.created_at` = `raw.altegio_appointments.date_time` = the APPOINTMENT date, NOT "record created" timestamp. Column name is misleading
- Breakdown: 590 confirmed + 36 cancelled_by_client. 356 have amount > 0 (€51,568 total confirmed revenue)
- **Pollution:** `funnels.py` (retention, LTV by source) does NOT filter `attendance_status='attended'` — future confirmed pollute cohort entry and retention. `cohorts.py` created a 2026-05 cohort from future bookings. `retention_monthly` has 1 row for May 2026
- Analytics tools that DO filter correctly: segmentation.py (14 attended filters), revenue.py (5 attended), client_ranking.py (6 attended)
- **FIX NEEDED:** Add `WHERE created_at <= CURRENT_DATE` AND/OR `attendance_status = 'attended'` in cohort/retention queries

**③ 29 future finance rows — ETL BUG**
- `finance_summary` built via `DELETE + INSERT GROUP BY date_trunc('day', created_at)` with `WHERE amount > 0` — catches future confirmed with amount > 0, GROUP BY creates date row, but `SUM(CASE WHEN attended...)` = 0 → phantom rows with $0 revenue
- All 29 future rows have service_revenue=0, total_revenue=0 — they're empty shells
- **FIX NEEDED:** Add `WHERE created_at <= CURRENT_DATE` or `HAVING SUM(CASE WHEN attended THEN amount ELSE 0 END) > 0` to finance_summary INSERT

### Analytics-mcp — BUGS FIXED (crash-test #3)
- ✅ FIXED: `VALID_ATTENDANCE_STATUSES` had `completed/cancelled/no_show/rescheduled` but DB has `attended/confirmed/cancelled_by_client/-1`
- ✅ FIXED: `segmentation.py` SQL syntax error in `high_deposit_inactive` (missing closing paren)
- ✅ FIXED: `champion` segment missing from `VALID_SEGMENTS`
- ✅ FIXED: `reactivation_orch.py` — JOIN uses `$2` → changed to `c.id` (correlate with outer query)
- ✅ FIXED: `secondary_services.py` — bare imports `from db import` consistent with server.py (NOT relative `.db`)
- REMAINING: 1,811 fact_appointment + 1,671 fact_deal rows with NULL client_id (pre-existing)
- REMAINING: `segment_clients_by_rules` excludes zero-visit clients (NULL >= N is NULL)
- REMAINING: `branch_id` ALL NULL, `source_id` ALL NULL, `is_first_visit` ALL false in fact_appointment
- REMAINING: ~1,893 fact_appointment rows have `service_id IS NULL`
- REMAINING: CAC = 0 everywhere (no marketing cost data)

### Crash-test #5 (2026-04-23) — FIXED BUGS
- ✅ FIXED: `get_client_activity_segment` segment_config used INNER JOIN for prospect/lead/unclassified → all returned 0 rows. Now uses LEFT JOIN with COALESCE. Also added `created_at <= CURRENT_DATE` to subqueries.
- ✅ FIXED: `inactive` removed from VALID_SEGMENTS (was passing validation then failing at segment_config lookup).
- ✅ FIXED: `lead` segment condition was `c.id < 0` (too broad, matched 16K). Now `c.id < 0 AND (visit_count = 0 OR visit_count IS NULL)`.
- ✅ FIXED: `get_retention_matrix` (funnels.py) didn't filter `attendance_status='attended'` → inflated cohort sizes. Added attended + CURRENT_DATE filters.
- ✅ FIXED: `get_ltv_by_source` (funnels.py) first_visit subquery didn't filter attended/CREATED_AT ≤ CURRENT_DATE.
- ✅ FIXED: `cohorts.py` both functions: `get_cohort_by_first_service` and `get_cohort_ltv` — no `attendance_status='attended'` filter → future bookings entered cohorts. Added attended + CURRENT_DATE.
- ✅ FIXED: `cohorts_advanced.py` — added `created_at <= CURRENT_DATE` to `get_cohort_by_first_service_category` and `evaluate_cohort_unit_economics` revenue CTE.
- ✅ FIXED: `revenue_daily` had 26 future-dated rows from old ETL. Deleted + added `DELETE WHERE date > CURRENT_DATE` to ETL before INSERT.
- KNOWN (not fixed): `client_activity_snapshot` missing lead/unclassified (only populates clients with appointments — by design).
- KNOWN (not fixed): 93 prospects have recency_days > 0 with 0 completed appointments — recency uses last booking date, not attended visit.

### Segmentation Model (updated 2026-04-23)

**Segments in `client_activity_snapshot`** (clients WITH appointments):
| Segment | Criteria | Count |
|---------|----------|-------|
| champion | 10+ visits, ≤30d recency, €500+ spend | 6 |
| active | ≤30d recency | 482 |
| at_risk | 31-90d recency | 36 |
| dormant | 91-180d recency | 25 |
| lost | >180d recency | 299 |
| prospect | 0 attended visits (only confirmed/cancelled) | 112 |

**Segments in `dim_client`** (ALL 24K clients):
| Segment | Source | Count |
|---------|--------|-------|
| unclassified | No visits, no conversations, no identity link | 19,141 |
| lost | From snapshot | 299 |
| active | From snapshot | 482 |
| lead | amoCRM contact with conversations but no Altegio link | 168 |
| prospect | Altegio client with bookings but 0 attended | 112 |
| at_risk | From snapshot | 36 |
| champion | From snapshot | 6 |
| dormant | From snapshot | 25 |
| inactive | (empty — no longer assigned) | 0 |

### ЗАВИСШИЕ Automation (amoCRM, 2026-04-23)

Pipeline 9476838 "Новые клиенты", source status 82208374 "Зависшие" → target statuses 85358010 "Зависшие (АВТО)" + 85358014 "ЗАВИСШИЕ (Ручная)".

**Key amoCRM v4 API bug:** `filter[status_id]` does NOT work for custom pipeline statuses — returns leads from default status 142 regardless. Must fetch ALL pipeline leads and filter locally by `status_id`.

**Logic:** Takes 40 unique leads from status 82208374, sorted by `updated_at` ASC (oldest first), requiring `updated_at` within last 120 days (relevance filter). Distributes first 20 to 85358010 (АВТО), next 20 to 85358014 (Ручная). Each moved lead gets tag "Зависшая".

| Component | Details |
|-----------|---------|
| Script | `scripts/stale_leads_automation.py` |
| State | `scripts/stale_leads_state.json` (moved_ids, batch_sizes, history) |
| Cache | `scripts/pipeline_leads_cache.json` (TTL=1h, 17,781 leads) |
| Cron | Hermes cron `853f65c92798` — 06:45 UTC = 08:45 Madrid |
| Tag | "Зависшая" added to each moved lead |
| Commands | `run`, `test`, `status`, `pause`, `resume`, `set-age <days>`, `set-count <status_id> <n>`, `history`, `reset-cache` |
| First run | 2026-04-23: moved 40 leads (20+20), 0 errors, 3860 remaining |
| Candidates | 3,900 leads updated within 120d in status 82208374 |

**Key ETL fix:** All mart queries now add `created_at <= CURRENT_DATE` when filtering attended appointments, preventing future bookings from inflating revenue, recency, and cohort calculations.

**retention_monthly fix (2026-04-23):** Old SQL: `cohort_size = COUNT(DISTINCT client_id)` per period_offset → varied per offset = always matched active_clients → 100% retention. New SQL: `cohort_sizes` CTE computes fixed FIXED size per cohort_month, then `period_activity` JOINs to compute `retention_rate = active_clients / cohort_size`. Result: realistic curves (Apr 2025: M1=58.4%, M2=39.2%, M3=29.3%).

## LLM Model Comparison (2026-04-23)

| Model | Avg Score/15 | Avg Time | Strengths |
|-------|-------------|----------|-----------|
| GLM-5.1 | 14.2 | 123s | Deeper analytics reasoning with ROI quant, faster code review, better SQL depth |
| Kimi-K2.6 | 14.5 | 126s | More subtle bug detection (5 vs 4 bugs), perfect Russian, concise, cleaner SQL structure |

**Per-test:**
- SQL: Tie (15/15 both) — both produce correct cohort retention queries
- Analytics: GLM wins (14 vs 13) — quantifies ROI with revenue math
- Code Review: Tie (15/15) — Kimi finds 1 extra bug but takes 2x longer (221s vs 122s)
- Russian: Kimi wins (15 vs 13) — perfect 3-sentence constraint compliance

**Recommendation:** GLM-5.1 as primary (analytics/SQL/reasoning), Kimi-K2.6 as secondary (code review, Russian). Kimi uses 2-4x more reasoning tokens = higher cost.

## DB Schema Column Name Traps

- client_rfm_scores: `segment` (NOT rfm_segment)
- client_segments: `name` (NOT segment_name)
- _sync_meta: `last_sync_at` (NOT last_synced)
- finance_summary: `date` (NOT period_date)
- fact_retail_sale: `total` (NOT amount)

## Meta Ads Account Reference

| ID | Name | Spent |
|----|------|-------|
| 1046675009730979 | Alef Med GrowUp | ~7.6M EUR |
| 2966847493508714 | ALEF MED new | ~25K EUR |
| 848891270106635 | new | ~5.4M EUR |
| 984537231195765 | Alef1 | 0 |

API calls need `act_` prefix. No subscribe or purchase action_type in this Meta account — only lead (2,068/month), offsite_conversion.fb_pixel_lead (2,056), onsite_web_lead (2,056). Custom conversions: empty. Pixel: "blv pixel". To track subscribe/purchase, need Meta Pixel/CAPI setup on the website.

## Daily Sync System (cron)

Runs at **22:00 UTC** (midnight Madrid) via crontab:
1. `sync_altegio.py` — pulls Altegio API data (clients, appointments, transactions) → raw → core → mart
2. `sync_amocrm.py` — pulls amoCRM API data (contacts, leads, events, notes) → raw → core transformations
3. `psql REFRESH MATERIALIZED VIEW mart.client_ltv_real`
4. `psql UPDATE mart.client_activity_snapshot SET segment = ...` (assign segments)
5. `psql UPDATE core.dim_client SET segment = ...` (sync segments to dim_client)
6. `restart_mcp_stack.sh` — restart analytics-mcp + openclaw-gateway
7. Telegram notification with stats

**Also runs:** `daily_report.sh` (08:00 UTC), `task_watchdog.sh` (every 30min), `crawl_alefmed.py` (04:00 UTC), `stale_leads_automation.py` (06:45 UTC)
**Log file:** `/var/log/analytics_sync.log`
**Lock file:** `/tmp/analytics_sync.lock` (flock-based, prevents concurrent runs)

### Bug-Fix Batch 2026-04-24 (10 critical → 5 combined fixes)

| Fix | Bug | File(s) | Change |
|-----|-----|---------|--------|
| 1 | `dim_client.first_visit_date` NULL for ALL 24K clients | sync_altegio.py + DB | Backfill 2,282 clients from `MIN(fact_appointment.created_at)`. Snapshot now uses `MIN(all appts)` not `MIN(attended)` |
| 2 | `recency_days` negative from future dates | sync_altegio.py | `GREATEST(CURRENT_DATE - ..., 0)` prevents negative. Segment CASE: `recency_days IS NULL OR < 0 → prospect`. Removed WHERE filter — all segments reclassified every sync |
| 3 | LTV phantom revenue (vcost vs vpaid) | ax2_cron.php | `firstRevenue`/`repeatRevenue` now use `vpaid()` (actual cash paid), not `vcost()` (list price max). LTV: `$firstRevenue + $repeatRevenue + ...` |
| 4a | daily_sync mkdir lock deadlock on SIGKILL/OOM | daily_sync_with_notify.sh | `mkdir` → `flock -n` (auto-released on process death). Hardcoded secrets → `source .env` |
| 4b | 429 rate-limit consumes retry budget for 5xx | sync_altegio.py, sync_amocrm.py | Separate budgets: 429 retries indefinitely with `Retry-After`, 5xx gets 3 attempts with exponential backoff |
| 4c | ax2_cron no flock + publicly accessible | ax2_cron.php + nginx | PHP flock guard. nginx `deny all` for `ax2_cron.php`. CLI `php -f` instead of `curl localhost` |
| 5 | ax2_cron overlaps with daily_sync (both ~22:00) | crontab | Offset to 23:30 UTC. `timeout 7200` guard |

**Segment distribution after fix:** lost=1040, dormant=397, at_risk=365, active=248, prospect=121, champion=111

### Cron Automation Audit (2026-04-24)

| # | Automation | Schedule | Status | Notes |
|---|-----------|----------|--------|-------|
| 1 | daily_sync_with_notify.sh | 22:00 UTC | ✅ OK | ~86min, all steps succeed |
| 2 | AX2 LTV Dashboard (ax2_cron.php) | 22:05 UTC | ✅ OK | ~466 clients, €149K LTV. BY_WEEK=true in cron. Dashboard default changed to by_week=1 (was 0). |
| 3 | crawl_alefmed.py | 04:00 UTC | ⚠️ FIXED | Was broken 5 days — DB_PASSWORD vs POSTGRES_PASSWORD mismatch in .env. Crawler reads DB_PASSWORD from analytics-mcp/.env, but .env only had POSTGRES_PASSWORD. **Fix:** added DB_PASSWORD=DB_HOST=DB_PORT=DB_NAME=DB_USER to analytics-mcp/.env. **Pitfall:** Python scripts using `load_dotenv()` may read different var names than what .env provides — always verify. |
| 4 | task_watchdog.sh | */30 min | ⚠️ FIXED | TELEGRAM_BOT_TOKEN not in cron env → all notifications silently failed. **Fix:** added `source .env` to script |
| 5 | daily_report.sh | 08:00 UTC | ⚠️ FIXED | Same TELEGRAM_BOT_TOKEN issue. **Fix:** added `source .env` to script |
| 6 | stale_leads_automation.py | 06:45 UTC | ⚠️ FIXED | Missing from crontab entirely! Only ran manually. **Fix:** added crontab entry |
| 7 | automation_watcher.sh | */15 min | ✅ NEW | Smart watchdog with auto-repair: restarts services, kills gateway orphans, checks SSE/PG/Ollama/TG. Dedup notifications. |

**Important pitfall:** Cron runs with minimal environment (no shell profile). If scripts reference env vars like `${TELEGRAM_BOT_TOKEN}`, they MUST `source /path/to/.env` or `export VAR=value` at the top. Without this, variables are empty and curl calls fail silently (output suppressed with `> /dev/null 2>&1`).

**Another pitfall:** When `.env` files use `POSTGRES_PASSWORD` but Python scripts look for `DB_PASSWORD`, `DB_HOST`, etc. — the vars don't match. Either add aliases OR change the script to use `POSTGRES_*` naming convention.

**openclaw-gateway restart loop:** If a stale process holds port 18789 outside systemd, the service enters a restart loop (counter reaching 700+). Fix: `kill` the stale PID, then `systemctl restart openclaw-gateway`.

### Sync Bug Patterns to Check
- **Column name mismatches** — raw table columns (e.g., `status_id`) may differ from core table columns (e.g., `stage_id`). Always `EXPLAIN INSERT` before running sync.
- **asyncpg type enforcement** — asyncpg validates Python types against Postgres column types at bind time. Strings passed to `date`/`timestamp` columns crash with `'str' object has no attribute 'toordinal'`. Must convert with `date.fromisoformat()` or `datetime.fromisoformat().date()`.
- **FK violations** — when INSERTing into tables with foreign keys (e.g., `fact_conversation_thread` → `dim_client`), ensure referenced rows exist first. Use `EXISTS` subqueries or populate parent tables before child tables.
- **Empty string defaults** — if raw data has empty strings (like `status_name = ""`), always provide a fallback (`COALESCE(dim_table.name, raw.field, '')`) or populate via JOIN with dimension tables.
- **COALESCE for ON CONFLICT UPDATE** — when updating parent rows that may have richer data (phone/email), use `COALESCE(EXCLUDED.phone, dim_client.phone)` to avoid overwriting existing data with NULLs.

- **Never use patch tool on .env files** — it corrupts secrets. Use `write_file` instead.
- **.env values may display truncated** — verify actual value lengths before assuming content
- **Gateway must restart after any MCP change** — `systemctl restart openclaw-gateway`
- **Meta API `act_` prefix** — account IDs like `1046675009730979` need `act_` prefix for API calls, BUT only for account-level endpoints. Campaign/ad/adset IDs do NOT get `act_` prefix. Using `act_` on campaign IDs causes 403 errors.
- **Meta API v21.0 field removals** — `message` field removed from AdCreative and Ad nested creative fields; `optimization_goal` removed from Campaign fields. Requesting these fields causes `(#100) Tried accessing nonexisting field` errors. Grep codebase for deprecated fields after Meta API upgrades.
- **Pydantic v2 model→string mismatch** — `validate("ClassName", ...)` passes string but function expects class reference
- **httpx token refresh** — `MetaAdsAPI.__init__` sets `self.headers` from env. Changing `api.access_token` does NOT update headers. Must recreate client or manually update `api.headers["Authorization"]`.
- **Meta API silent empty data** — some invalid tokens return `{"data": []}` with HTTP 200 instead of error. Check for empty data as potential auth issue.
- **amoCRM MCP search implementation** — `@theyahia/amocrm-mcp` v2.0.2 search tool was broken (called non-existent `/api/v4/search`). Local patched fork at `amocrm-mcp-patched/` uses parallel `/leads`, `/contacts`, `/companies`. Zod schemas enforce `.int().min(1)` on limit params. Always use local patched path, NOT npx.
- **Config token truncation trap** — `read_file` truncates long values (like 1103-char JWT tokens). When patching config.yaml, always verify the actual file content with `cat` or `wc -c` — never trust the displayed truncated value. If patching replaces a long secret with its truncated display form, the secret is corrupted.
- **Analytics MCP try/except pattern** — all 81 async tool functions across 18 files use `try/except Exception as e: return {"error": str(e)}`. When adding new tools, always wrap in try/except. — analytics-mcp uses `get_pool()` (not `init_pool()`). Call `await get_pool()` first.
- **DB connection from scripts** — use the same venv and env loading mechanism as the running systemd service for consistent credential resolution
- **f-string JSON path trap** — PostgreSQL `#>> '{key,path}'` inside Python f-strings creates `{key,path}` format expression → `NameError`. Must always escape to `#>> '{{key,path}}'`. Grep for: `grep -rn "#>> '" --include='*.py'` then check if line is inside f-string (look for `f"""` above it). TWO copies of validation.py exist (root + tools/) — always fix BOTH.
- **negative client_ids** — amoCRM contacts stored as negative IDs in dim_client (e.g., -44766261). `validate_client_id()` must allow non-zero integers, not just positive.
- **asyncpg string→date crash** — asyncpg validates Python types at bind time. Strings like `'1991-09-01'` passed to DATE columns crash with `'str' object has no attribute 'toordinal'`. Always convert with `date.fromisoformat()` + fallback to `datetime.fromisoformat().date()`. Also handle `datetime` objects (`.date()`).
- **fact_appointment.created_at is NOT a creation timestamp** — it maps to `raw.altegio_appointments.date_time` which is the **appointment date** (when the visit is scheduled), NOT when the record was created. Future appointments (status=confirmed) have future `created_at` values. All mart queries must add `created_at <= CURRENT_DATE` when computing historical analytics. The Altegio API pull uses `end_date = now + 30 days` to sync upcoming bookings.
- **Column name mismatches between raw and core** — raw tables (e.g., `status_id`) may map to differently-named core columns (e.g., `stage_id`). Always `EXPLAIN INSERT` against actual DB schema before running sync.
- **Cron env has no shell profile** — cron runs with minimal environment. Scripts using `${TELEGRAM_BOT_TOKEN}` or other env vars MUST explicitly `source /path/to/.env` or export vars. Without this, variables are empty and curl calls fail silently when output is suppressed with `> /dev/null 2>&1`.
- **DB_PASSWORD vs POSTGRES_PASSWORD .env mismatch** — `crawl_alefmed.py` loads `analytics-mcp/.env` via `load_dotenv()` and reads `DB_PASSWORD`, `DB_HOST`, etc. But .env only contained `POSTGRES_PASSWORD`, `POSTGRES_HOST`, etc. The `DB_*` vars defaulted to empty → `InvalidPasswordError`. Always verify which env var names a script reads vs what .env provides.
- **Crontab entries can disappear** — the ЗАВИСШИЕ automation script had no crontab entry and only ran via Hermes cron or manually. Always verify `crontab -l` matches expected schedule after config changes.
- **Crontab overlap** — two jobs at the exact same minute (e.g., daily_sync and ax2_cron both at `0 22 * * *`) cause resource contention. Stagger by 90+ min (e.g., sync 22:00, ax2 23:30) since sync can take 60-90 min and hits the same Altegio API.
- **openclaw-gateway restart loop from duplicate systemd service** — ROOT CAUSE: `openclaw gateway start` CLI creates a user-level systemd service at `~/.config/systemd/user/` that competes with the system-level service. Both bind port 18789 and kill each other via SIGTERM. Fix: disable user-level service. Also: CIAO bonjour mDNS library crash (`CIAO PROBING CANCELLED`) requires `--unhandled-rejections=none` in ExecStart. Without it, process exits 1 on mDNS name conflict.
- **Telegram bot menu commands** — openclaw.json `customCommands` array defines `/command` entries shown in bot menu. Update the array, then restart gateway. Bot uses polling mode (no webhook).
- **Telegram bot diagnostics** — check with `getMe`, `getWebhookInfo`, `getUpdates` API methods. Chat ID for notifications in `.env`. Use `parse_mode=Markdown` for formatted messages.
- **Gateway systemd service** — `/etc/systemd/system/openclaw-gateway.service` is the ONLY service that should run. If `openclaw gateway start` is ever run manually (or by OpenClaw's own setup), it creates a SECOND user-level service at `/root/.config/systemd/user/openclaw-gateway.service`. **These two compete for port 18789, sending SIGTERM to each other.** This was the root cause of constant gateway crashes (observed restart counter=627, SIGTERM every ~23s).
- **Fix (2026-04-27):** (1) Disabled user-level service: `systemctl --user disable openclaw-gateway && mv ~/.config/systemd/user/openclaw-gateway.service ~/.config/systemd/user/openclaw-gateway.service.disabled`. (2) Added `--unhandled-rejections=none` to ExecStart (prevents CIAO bonjour mDNS crash: `Unhandled promise rejection: CIAO PROBING CANCELLED`). (3) `Restart=always` (not `on-failure` — with the node flag, process exits 0 on SIGTERM but `on-failure` only restarts on non-zero). (4) `ExecStartPre=-/usr/bin/killall -9 openclaw-gateway` + `ExecStopPost=-/usr/bin/killall -9 openclaw-gateway` to clean orphans before start and after stop. (5) `RestartSec=15` for faster recovery.
- **CIAO PROBING CANCELLED** — bonjour/mDNS library (ciao) throws unhandled promise rejection when service name conflicts during probing. Without `--unhandled-rejections=none`, Node.js kills the process with exit code 1. This happens on every gateway start after ~15s.
- **WARNING:** Running `openclaw gateway start` or `openclaw gateway stop` re-creates the user-level service file. After any `openclaw` CLI gateway command, check and disable it again.

**automation_watcher.sh** — `/root/.openclaw/workspace/scripts/automation_watcher.sh` (crontab */15). Checks: 5 services, gateway restart-loop (NRestorts>10), EADDRINUSE in recent logs, 3 SSE endpoints (:8101/:8102/:8103), PostgreSQL (connectivity + long queries), sync freshness (>26h), disk (>90% auto-cleanup), RAM (>90%), Ollama (:11434), Telegram API. Auto-repairs: restart down services, kill gateway orphans + clean start, PostgreSQL restart, disk cleanup (tmp, logs, journal vacuum). Dedup via md5 state file — only notifies when problem state changes. Logs repairs to separate `repairs.log`. Uses `set -uo pipefail`.

**Pitfall:** `set -u` in bash scripts + associative array iteration: `for svc in "${!SERVICES[@]}"` works, but function args like `check_service "$svc" "$friendly"` with `set -u` fail if `friendly_name` is declared but the caller only passes 1 arg. Removed unused `$2` param.

**Pitfall:** `grep -c` inside `$()` with `set -u` — if grep finds nothing on some systems, returns multi-line or empty. Use `${VAR:-0}` + `tr -d '[:space:]'` + `-n` check before integer comparison.

**Pitfall:** PostgreSQL password in .env is `POSTGRES_PASSWORD`, not `DB_PASSWORD`. Automation watcher uses `PGPASSWORD="${POSTGRES_PASSWORD:-}"` from sourced .env.

**Telegram bot commands** — Update BOTH: (1) `openclaw.json` `customCommands` array, (2) Telegram `setMyCommands` API. openclaw.json defines what OpenClaw gateway exposes; setMyCommands defines what users see in the Telegram menu. They should stay in sync. Current 21 commands include analytics (/ltv, /funnel, /cohort, /retention, /segments, /top, /reactivation, /conversations, /revenue), CRM (/leads, /search, /ads), system (/status, /sync, /restart, /health, /watchdog, /bugs), session (/new, /help).
**⚠️ Telegram polling is UNSTABLE in OpenClaw 2026.4.x** — grammy runner rebuilds transport every 30s + cluster mode 2-process conflict = periodic 409 errors + 0 incoming messages. Priority fix: migrate to webhook mode (needs HTTPS endpoint via Cloudflare Tunnel or Let's Encrypt + nginx reverse proxy).

**Adding Ollama cloud models** — add to 3 places: (1) `openclaw.json` models.providers.ollama.models array, (2) openclaw.json agents.defaults.models dict (with `ollama/` prefix), (3) Hermes `config.yaml` providers section (with base_url + model + provider). Model ID format: `model-name:cloud` (Ollama), `ollama/model-name:cloud` (OpenClaw).
- **Telegram bot menu commands** — use `setMyCommands` API with the full bot token (46 chars, not the masked/truncated version). Shell scripts mask tokens on output — get full token via `source .env && echo $TELEGRAM_BOT_TOKEN`.
- **amoCRM POST /leads/tags requires array** — `{"name": tag_name}` returns 400 "should be of type array". Must use `{"name": [tag_name]}`. Fixed in `amo_automation/amo_client.py:283`.
- **automation_watcher gateway cascade fix (2026-04-24)** — Watcher restarted gateway every 15 min unnecessarily because `systemctl is-active` returned false during transport rebuild (brief down). Caused 1903 restarts/day. Fix: (1) Cooldown file `/tmp/watcher_restart_${svc}` — don't restart if last restart <300s ago. (2) For gateway specifically: double-check after 10s sleep before restarting. (3) Gateway restart-loop detection only triggers if gateway is CURRENTLY DOWN (not just high NRestarts counter). (4) systemd StartLimitBurst=10/600s (was 3/300s), StartLimitAction=none. (5) When gateway is UP but NRestarts>10, just `reset-failed` counter silently.
- **Telegram token mismatch trap** — openclaw.json had `botToken` from @leadbitqa_bot (7427361380) while .env had token for @anastasialeadbot (7701224581). Symptoms: 0 incoming messages, getUpdates conflict. telegram-offset.json also stored wrong botId. **Fix:** update botToken in openclaw.json AND reset telegram-offset.json with correct botId then restart gateway.
- **Telegram diagnostics — DO NOT call getUpdates from CLI!** Running `curl /bot{token}/getUpdates` from any process (including Hermes agent testing) CANCELS the active long-polling cycle in the gateway → 409 Conflict. Use `getWebhookInfo` and `getMe` instead (read-only, don't interfere with polling).
- **Gateway orphan process kill sequence** — When gateway won't restart cleanly: (1) `systemctl stop openclaw-gateway`, (2) `killall -9 openclaw-gateway`, (3) verify `pgrep -a openclaw-gateway` returns nothing, (4) `systemctl reset-failed`, (5) `systemctl start`. Step 3 is critical — orphan processes hold port 18789 and systemd spawns new ones that hit EADDRINUSE.
- **Gateway user-level service conflict** — The #1 cause of gateway instability. `openclaw gateway start` creates `/root/.config/systemd/user/openclaw-gateway.service` which runs alongside `/etc/systemd/system/openclaw-gateway.service`. Both bind port 18789 and send SIGTERM to each other every ~20-25s. **Always check `systemctl --user status openclaw-gateway`** if gateway keeps crashing. Disable with `systemctl --user disable openclaw-gateway && mv ~/.config/systemd/user/openclaw-gateway.service{,.disabled}`.
- **mkdir lock → flock** — `mkdir`-based locks deadlock when process is killed (SIGKILL/OOM) because `rmdir` trap never fires. Use `flock -n` instead: `exec 200>"$LOCKFILE"; if ! flock -n 200; then exit 0; fi`. flock is released by kernel on process death.
- **dim_client dimension backfill pattern** — When adding a new dimension column (e.g., `first_visit_date`), always backfill from fact tables: `UPDATE dim_client dc SET col = sub.val FROM (SELECT client_id, MIN(created_at)::date AS val FROM fact_table WHERE client_id IS NOT NULL GROUP BY client_id) sub WHERE sub.client_id = dc.id AND dc.col IS NULL`. Verify with `COUNT(col) vs COUNT(*)`.
- **Snapshot first_visit_date must be MIN(all appts) not MIN(attended)** — `MIN(created_at) FILTER (WHERE attendance_status='attended')` returns NULL for clients who booked but never showed. Use `MIN(created_at) FILTER (WHERE created_at <= CURRENT_DATE)` instead to capture first touch point.
- **segment CASE must reclassify ALL rows every sync** — Old code: `WHERE segment IS NULL OR segment IN ('inactive','prospect')` missed transitions (active→at_risk). Remove the WHERE clause so UPDATE applies to all rows. Add guard: `WHEN completed_appointments = 0 THEN 'prospect'` before recency-based rules.
- **recency_days negative guard** — Future appointments make `CURRENT_DATE - future_date` negative. Wrap in `GREATEST(..., 0)`. Also add `WHEN recency_days IS NULL OR recency_days < 0 THEN 'prospect'` to segment CASE.
- **daily_report.sh same .env pattern** — All scripts (daily_sync, watchdog, daily_report) must `source .env` for secrets. Never hardcode PGPASSWORD or TELEGRAM_BOT_TOKEN. `.env` uses `POSTGRES_PASSWORD` (not `DB_PASSWORD`).
- **Automated crash-test cronjob** — Hermes cronjob `4bdc94c3a152` runs daily at 12:00 UTC, rotates between glm-5.1 and kimi-k2.6:cloud via `/root/.hermes/cron/crash_test_state.json`. Rotation script at 11:55 UTC. Delivers report to Telegram.
- **429 vs 5xx retry budget** — API 429 (rate-limit) should NOT consume retry attempts meant for 5xx server errors. Use `while retries_5xx < MAX` loop: 429 does `continue` without increment, 5xx increments counter. 429 reads `Retry-After` header.
- **vcost vs vpaid in LTV** — `vcost()` = max(cost, manual_cost, first_cost) = list/expected price. `vpaid()` = cost field only = actually charged. For revenue/LTV reporting, always use vpaid (cash actually paid). Using vcost inflates revenue (observed: ~€60K phantom revenue).
- **nginx deny for cron scripts** — PHP scripts meant only for cron should be blocked from direct web access: `location ~ ^/script\.php$ { deny all; }`. Run via CLI `php -f` instead of `curl localhost`.
- **php flock pattern** — `$lockFile = fopen('/tmp/name.lock', 'c'); if (!flock($lockFile, LOCK_EX | LOCK_NB)) { exit(0); } register_shutdown_function(function() use ($lockFile) { flock($lockFile, LOCK_UN); fclose($lockFile); });`
- **snapshot segment CASE must reclassify all** — `UPDATE ... SET segment = CASE ... END` without `WHERE` filter. Old code had `WHERE segment IS NULL OR = '' OR = 'inactive' OR = 'prospect'` which missed segment transitions (e.g., active→at_risk when recency increased).** — prefer system crontab for reliability (persistent, logged, visible). Hermes cronjobs are ephemeral and can be lost. If using both, ensure no duplicates. All Hermes cronjobs removed 2026-04-24 — only system crontab now.
- **Telegram bot setMyCommands** — use the full bot token (46 chars from .env, not the masked/truncated version). Shell scripts mask tokens — get full token via `source .env && echo $TELEGRAM_BOT_TOKEN`.
- **AX2 BY_WEEK default** — `index.php` line 30 now defaults by_week to '1' (was '0'), so weekly breakdown shows on first visit without toggling. `ax2_cron.php` already had BY_WEEK=true.
- **Hermes busy_input_mode** — set to `queue` (was `interrupt`) in config.yaml so user messages don't kill running tasks.
- **Wazzup24 v3 message API format** — `send_waba_template_to_client` must use `{channelId, chatType: "whatsapp", chatId: "<phone>", text: "...", templateName: "...", templateVariables: {...}}`. Sending `phone` key instead of `chatId`+`chatType` causes 400 INVALID_MESSAGE_DATA.
- **Wazzup24 api_post error structure** — `api_post` returns `{"error": "HTTP 400", "status_code": 400, "body": {...}}` on API errors. When surfacing to user, extract the real error from `body.message` or `body.error` instead of just showing "HTTP NNN".
- **Wazzup24 empty phone guard** — `send_whatsapp_message` and similar must validate `phone` is non-empty and contains digits after stripping non-digit chars before making the API call.
- **Wazzup24 XSS in template preview** — `preview_waba_template` must `html.escape()` both `template_name` and variable values to prevent XSS injection in preview output.
- **Wazzup24 channelId vs channel type** — `channelId` in Wazzup24 v3 API must be a UUID from GET /channels, NOT a type string like "whatsapp". Always validate that channel IDs look like UUIDs before sending. `send_waba_template_to_client` now validates this and returns a clear error if a type string is passed.
- **Meta Ads daily_budget must be string** — Meta Graph API requires `daily_budget` as a string (e.g., `"5000"`), not an integer. `create_ad_set` already does `str()`, but `create_campaign`, `update_campaign`, and `update_ad_set` all passed it as int. Always stringify budget values.
- **Meta Ads duplicate_campaign response** — POST `{campaign_id}/copies` returns `{"copied_campaign_id": "..."}`, NOT `{"id": "..."}`. Use `copied_campaign_id` key.
- **Meta Ads get_creative response** — `{ad_id}/creative` edge returns `{"data": [...]}` (array), not a single dict. Must extract `response["data"][0]`.
- **Meta Ads create_ad without creative** — silently creates ad in invalid state (can never be activated). Always require `creative_id`.
- **Meta Ads get_insights level validation** — `level` param only accepts `account`, `campaign`, `adset`, `ad`. Invalid levels silently return empty results. Now validated at both tools and API layer.
- **Meta Ads targeting dict mutation** — `create_ad_set` was mutating caller's `targeting` dict in-place with `targeting["targeting_automation"] = ...`. Now creates a copy: `{**targeting, ...}`.
- **Wazzup24 template validation** — `validate_waba_template_variables` must accept `template_body` param (the actual template content), not just `template_name`. Template names like "hello_world" don't contain `{{1}}` placeholders.
- **Wazzup24 api_post/api_patch dead code** — `raise_for_status()` after an early `return` on status >= 400 is unreachable. Don't add it back.
- **Wazzup24 get_unanswered_counter** — without user_id, now calls `/unanswered` (all users) instead of hardcoded user `12364798`.
- **Wazzup24 scope='card' without filter** — `get_chat_window_link` now returns explicit error instead of silently dropping required params.
- **Wazzup24 connection pooling** — uses shared `httpx.AsyncClient` via `_get_client()` instead of creating new client per request.
- **amoCRM falsy check `if (params.X)` drops 0** — In JS, `0` is falsy so `page=0` defaults to page 1, `limit=0` is dropped. Fixed to `if (params.X != null)` across 6 files.

### Crash-test #5 Results (2026-04-24) — 8 BUGS FOUND + FIXED

| # | Severity | Bug | File | Fix |
|---|----------|-----|------|-----|
| 1 | 🔴 CRITICAL | `get_client_activity_segment` segment_config used INNER JOIN → prospect/lead/unclassified ALWAYS returned 0 rows | segmentation.py:209 | LEFT JOIN + COALESCE + CURRENT_DATE |
| 2 | ⚠️ MEDIUM | `inactive` in VALID_SEGMENTS but not in segment_config → "Unknown segment" error | validation.py (2 copies) | Removed from set |
| 3 | ⚠️ MEDIUM | `lead` condition `c.id < 0` → matched 16K instead of 168 | segmentation.py:185 | Added `AND (visit_count = 0 OR visit_count IS NULL)` |
| 4 | 🔴 HIGH | `get_retention_matrix` no `attendance_status='attended'` → inflated cohorts with future/cancelled | funnels.py:68-83 | Added attended + CURRENT_DATE filters |
| 5 | ⚠️ MEDIUM | `get_ltv_by_source` first_visit subquery no attended filter | funnels.py:137 | Added attended + CURRENT_DATE |
| 6 | 🔴 HIGH | `cohorts.py` both functions missing attended → future bookings in cohorts | cohorts.py:32,113 | Added attended + CURRENT_DATE |
| 7 | ⚠️ MEDIUM | `cohorts_advanced.py` missing CURRENT_DATE | cohorts_advanced.py:103,361 | Added `created_at <= CURRENT_DATE` |
| 8 | ⚠️ MEDIUM | `revenue_daily` 26 future rows from old ETL runs | sync_altegio.py + DB | DELETE + ETL cleanup before INSERT |

**Known (not fixed, documented):**
- `client_activity_snapshot` missing lead/unclassified (only populates clients with appointments — by design)
- 93 prospects have recency_days > 0 with 0 completed appointments (recency uses booking date)
- 1,811 orphaned appointments (NULL client_id)
- retention_monthly vs cohort_ltv offset=0: diff=112 (different filter sources)

### amo_automation Toolkit (2026-04-23)

**Location:** `/root/.openclaw/workspace/openclaw-mcp-servers/amo_automation/`

| Module | Purpose |
|--------|---------|
| `amo_client.py` | AmoCRM v4 API client: GET/POST/PATCH, pagination, rate limit (0.4s), file cache (1h TTL) |
| `notifier.py` | Telegram HTML notifications with auto-chunking (4000 char) |
| `state.py` | JSON state: pause/resume, batch_sizes, moved_ids dedup (10K cap), history (30 cap) |
| `pipeline.py` | Pipeline helper: get_candidates (filter by age+dedup), distribute (N leads → M targets), move_and_tag, build_report |
| `cli.py` | CLI framework: run/test/status/pause/resume/set-count/set-age/history/reset-cache |
| `generator.py` | Scaffolding: `python3 -m amo_automation.generator create --name "..." --pipeline ID --source ID --targets ID:count ...` |

**ЗАВИСШИЕ v2** migrated to toolkit (30-line constructor). Cron: Hermes cron `853f65c92798` 06:45 UTC = 08:45 Madrid.
**First run (2026-04-23):** 40 leads moved (20+20), 0 errors, 3860 remaining candidates.
**amoCRM API bug:** `filter[status_id]` does NOT work for custom pipeline statuses. Must fetch ALL pipeline leads and filter locally.
**Logic:** 40 unique leads from status 82208374, sorted by updated_at ASC (oldest first), updated_at within 120d. First 20 → 85358010, next 20 → 85358014. Tag "Зависшая" on each.
- **amoCRM `.catch(() => ({}))` swallows errors** — Changed to log errors with `console.error()` instead of silently returning empty results.
- **amoCRM search total** — Now uses `_total_items` from API response when available for accurate total count.
- **amoCRM `response.json()` crash on empty body** — Wrapped in try/catch, returns `{}` on parse failure.
- **amoCRM parseInt Retry-After** — Handles HTTP-date format strings that would produce NaN.
- **Analytics MCP imports** — tool files use bare imports (`from db import`) NOT relative imports (`from .db import`). The server.py runs from the project root and adds `tools/` to sys.path, making bare imports work. Relative imports cause `ModuleNotFoundError: No module named 'tools.db'`.
- **systemd restart for bug fixes** — `systemctl kill analytics-mcp --signal=SIGHUP` triggers a restart. Plain `systemctl restart` may time out in tool calls.
- **⚠️ PATCH TOOL INDENTATION TRAP** — When using the `patch` tool on Python files, multi-line replacements can lose indentation, causing `IndentationError` at runtime. This happened TWICE in crash-test #4: (1) `app.py` repeat_pct fix lost indent on 2 lines, (2) `meta_ads.py` cid removal lost indent on 1 line. **ALWAYS** run `python3 -c "import py_compile; py_compile.compile('/path/to/file.py', doraise=True)"` after patching Python, then restart the service. The linter check in patch tool does NOT catch IndentationError.
- **restart_mcp_stack.sh gateway timing** — openclaw-gateway takes 15-20 seconds to bind port 18789 (loads plugins, auth, SSE). The script's original `sleep 8` caused false ❌ in daily sync reports. Fixed with retry loop (6 attempts × 5s = 30s max wait).
- **fact_appointment.created_at is MISLEADING** — it's actually the appointment date (`date_time` from Altegio API), NOT the record-creation timestamp. The raw table has both `date_time` (appointment date) and `created_at` (booking date). The core INSERT maps `date_time → created_at`. Always remember: `created_at` in fact_appointment = when the appointment IS scheduled, not when the row was written. Future appointments stored with future created_at values.
- **client_activity_snapshot recency_days negative** — formula `COALESCE(MAX(created_at) FILTER (WHERE attended), MIN(created_at))` falls back to MIN(created_at) for clients with 0 attended visits. If their only appointments are future confirmed, MIN is a future date → negative recency_days → falsely classified as "active". Fix: use `MIN(created_at) FILTER (WHERE created_at <= CURRENT_DATE)` as fallback.
- **finance_summary phantom rows** — DELETE+INSERT with GROUP BY date creates rows even when SUM = 0. Future appointments with amount > 0 but status confirmed (not attended) create date rows with $0 revenue. Fix: add `WHERE created_at <= CURRENT_DATE` or `HAVING` clause.
- **"inactive" segment = default bucket for ALL clients without visits** — 80.4% of dim_client. Includes 13,690 amoCRM contacts without identity_map link. Should differentiate: lead (amo no link), prospect (Altegio no visits), inactive (had visits >180d ago). Daily sync line 122-125: `SET segment='inactive' WHERE segment IS NULL OR segment = ''`
- **Future data in retention/cohort calculations** — `funnels.py` and `cohorts.py` don't filter `attendance_status='attended'` or `created_at <= CURRENT_DATE`. Future confirmed bookings create phantom cohort entries (e.g., May 2026 cohort with 0 real visits).

## Web UI — Nginx & External Access

Open WebUI runs in Docker on **port 3000**. Nginx reverse-proxies on **port 3001**.

**Critical lesson:** nginx config was symlinked to `/tmp/open-webui-nginx.conf` which got wiped on reboot (files in `/tmp` don't survive). Always place configs in `/etc/nginx/sites-available/` with permanent symlinks from `sites-enabled/`.

- Config: `/etc/nginx/sites-available/open-webui` (listen 3001 → proxy_pass localhost:3000)
- Symlink: `/etc/nginx/sites-enabled/open-webui` → `/etc/nginx/sites-available/open-webui`
- Must include WebSocket support (`/ws` location with upgrade headers) and SSE support (`proxy_buffering off`, `proxy_http_version 1.1`, `Connection ""`)
- **Verify nginx after changes:** `sudo nginx -t && sudo systemctl restart nginx`
- **External URL:** `http://83.138.53.219:3000` (direct Docker) or `http://83.138.53.219:3001` (nginx proxy)

Other web services externally accessible on `83.138.53.219`:
| Port | Service | Notes |
|------|---------|-------|
| 3000 | Open WebUI (Docker) | Direct access, no proxy |
| 3001 | Open WebUI (nginx) | Reverse proxy with WS/SSE |
| 8501 | Streamlit Dashboard | Analytics dashboard |
| 8080 | AX2 PHP Dashboard | LTV by procedure, Altegio Analytics |

**AX2 endpoints:**
- `http://83.138.53.219:8080/ax2_load.php` — main dashboard (loads cached data)
- `http://83.138.53.219:8080/ax2_cron.php` — cache regeneration (runs via cron daily at 22:00 UTC)
- `http://83.138.53.219:8080/index.php` — main index
- **Cron:** `0 22 * * * curl -s http://localhost:8080/ax2_cron.php --max-time 1800 >> /var/www/ax/logs/cron.log 2>&1`
- **Dependencies:** `php8.3-mbstring` required for `mb_strtolower()` in `isDiag()` function — installed Apr 2026
- **Cache:** `/var/www/ax/cache.json` (~740KB, 1075 candidates)

UFW firewall is **inactive** — all ports open.

## Altegio Integration (NOT a separate MCP)

Altegio (formerly Yclients) data flows via ETL script, not an MCP server:
- Script: `scripts/sync_altegio.py` — pulls clients, appointments, services, staff from Altegio API
- Config: `ALTEGIO_COMPANY_ID=1285692`, `ALTEGIO_API_TOKEN`, `ALTEGIO_USER_TOKEN` in shared `.env`
- Data flow: Altegio API → raw_altegio_* tables → core → mart schemas in PostgreSQL
- Auth header: `Bearer {API_TOKEN}, User {USER_TOKEN}` with `Accept: application/vnd.api.v2+json`

## Wazzup24 v3 API — Verified Endpoints

- OK: GET /v3/channels, GET/PATCH /v3/webhooks, GET /v3/unanswered/{userId}, POST /v3/message
- DOES NOT EXIST: /v3/templates, /v3/messages/send (use /v3/message), /v3/messages/{id}/status

### Wazzup24 v3 Message Format (POST /v3/message)

**REQUIRED fields:** `channelId`, `chatId`, `chatType`, `text`
- `channelId` — UUID from GET /v3/channels (e.g., `16913fcb-44c3-4a6e-8d33-a64ebac6e43e`)
- `chatId` — phone number in international format WITHOUT + (e.g., `380509001800`)
- `chatType` — always `"whatsapp"`
- `text` — message text
- ❌ WRONG: sending only `channelId` + `phone` → 400 INVALID_MESSAGE_DATA (missing chatId, chatType)
- ✅ CORRECT: `{"channelId": "UUID", "chatId": "380509001800", "chatType": "whatsapp", "text": "Hello"}`

### amoCRM MCP — Details (npx @theyahia/amocrm-mcp v2.0.2)

- 19 tools: list/get/create/update leads, list/get/create contacts, list/create companies, list/create/complete tasks, list pipelines, add_note, search, list/accept unsorted, list_events, get_account
- Subdomain: nkataeva07gmailcom. API base: `https://nkataeva07gmailcom.amocrm.ru/api/v4`
- Now in BOTH openclaw.json AND Hermes config.yaml (added Apr 2026)
- AMO API v4 pagination: no total count returned, use `_links.next` presence to check for more pages, 250 per page max
- **Custom fields for birthday:** field_id 974853 ("День рождения", type=birthday) + field_id 994013 ("Fecha de nacimiento", type=date). 1,988 contacts have BOTH fields. 2,326 unique contacts have any birth date out of 16,086 total (14.5%)
- **Custom fields endpoint:** GET /api/v4/contacts/custom_fields — returns all 18 contact custom fields
- **Filter contacts by custom field:** GET /api/v4/contacts?with=custom_fields_values — returns all fields inline; direct filter by field value has tricky syntax, simpler to fetch all and filter client-side

## Analytics MCP Tool Categories (70 tools total)

| Category | Module | Example Tools |
|----------|--------|---------------|
| Revenue & Finance | revenue.py | get_revenue, get_finance_summary, compare_periods |
| Clients | clients.py | get_client_360, get_new_vs_repeat, get_clients_rfm_scores |
| Appointments | appointments.py | get_appointments_summary |
| Funnels | funnels.py | get_sales_funnel, get_retention_matrix, get_ltv_by_source |
| Cohorts | cohorts.py | get_cohort_by_first_service, get_cohort_ltv |
| Advanced Cohorts | cohorts_advanced.py | get_cohort_by_first_service_category, evaluate_cohort_unit_economics |
| System | system.py | get_metric_definition, get_data_freshness, get_sync_status |
| Identity | identity.py | get_identity_overview, get_client_unified, get_reactivation_candidates |
| Reports | reports.py | get_report_summary |
| Conversations | conversations.py | get_conversation_stats, get_client_conversations, get_response_time_analysis |
| Reactivation | reactivation.py | generate_reactivation_messages, preview_reactivation_campaign |
| Wazzup24 proxy | wazzup24.py | handle_get_channels, handle_get_templates, handle_send_message |
| Segmentation | segmentation.py | rank_clients_by_metric, get_vip_clients, get_at_risk_clients, get_lost_clients |
| Conversation AI | conversation_ai.py | classify_client_intent, extract_requested_procedure, extract_objections |
| Reactivation Orch | reactivation_orch.py | personalize_message, create_reactivation_queue, rank_reactivation_priority |
| Secondary Services | secondary_services.py | get_secondary_services, get_secondary_services_matrix |
| Client Ranking | client_ranking.py | rank_clients_by_value, get_client_value_profile |
| Messaging | messaging.py | log_message, get_message_log, get_outreach_analytics |
| Validation | validation.py | (internal helper) |

### amo_automation Module (2026-04-23)

Reusable Python toolkit at `amo_automation/` for building pipeline automations.

| Module | Purpose |
|--------|---------|
| `amo_client.py` | AmoCRM v4 API client: GET/POST/PATCH, pagination, rate limiting (0.4s), file cache (1h TTL), pipeline/status helpers |
| `notifier.py` | Telegram HTML notifications with auto-chunking (4000 char) |
| `state.py` | JSON state manager: pause/resume, batch sizes, moved_ids dedup (10K cap), history (30 cap) |
| `pipeline.py` | Pipeline helper: get_candidates (filter by age+dedup), distribute (N unique leads → M targets), move_and_tag, build_report |
| `cli.py` | CLI framework: run/test/status/pause/resume/set-count/set-age/history/reset-cache |
| `generator.py` | Scaffolding: `python3 -m amo_automation.generator create --name "..." --pipeline ID --source ID --targets ID:count ...` |

**Creating a new automation:**
```bash
python3 -m amo_automation.generator create \
  --name "Новая автоматизация" \
  --pipeline 9476838 --source 82208374 \
  --targets 85358010:20 85358014:20 --tag "МойТег" --age 120
``` @theyahia/amocrm-mcp v2.0.2)

Leads: list_leads, get_lead, create_lead, update_lead, search
Contacts: list_contacts, get_contact
Companies: list_companies, create_company
Tasks: create_task, complete_task, list_tasks
Notes: add_note
Pipeline/Account: list_pipelines, get_account, list_events, list_unsorted, accept_unsorted