---
name: amocrm-automation
description: amoCRM sync and automation — sync_amocrm.py, sync_amo_events.py, sync_birthdays_to_amo.py, amo_client.py. Contact/lead/event sync, stale leads, birthday push.
version: 1.0
---

# amoCRM Automation Skill

## Overview
Scripts for syncing amoCRM data into PostgreSQL and automating lead management.

## File Locations
- `sync_amocrm.py`: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_amocrm.py` (477 lines)
- `sync_amo_events.py`: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_amo_events.py`
- `sync_birthdays_to_amo.py`: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_birthdays_to_amo.py`
- `amo_client.py`: `/root/.openclaw/workspace/openclaw-mcp-servers/amo_automation/amo_client.py` (306 lines)
- Venv: `/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/python3`
- Env: `/root/.openclaw/workspace/openclaw-mcp-servers/.env`

## AmoClient (amo_client.py)

### Key Methods
| Method | Description |
|--------|-------------|
| `get(path, params)` | GET request with auto-throttle |
| `post(path, data)` | POST request |
| `patch(path, data)` | PATCH request |
| `get_all_pages(path, params)` | Paginated GET with caching |
| `get_pipelines()` | Get all pipelines |
| `get_pipeline_statuses(pipeline_id)` | Get statuses in pipeline |
| `get_all_pipeline_leads(pipeline_id)` | All leads in pipeline (cached) |
| `get_leads_in_status(pipeline_id, status_id)` | Leads in specific status |
| `move_leads(leads_with_target)` | Move leads between statuses |
| `ensure_tag(tag_name)` | Create tag if not exists (**note: requires array format**) |
| `get_lead(lead_id)` | Get single lead details |
| `add_note(lead_id, text)` | Add note to lead |

### Critical: ensure_tag
amoCRM API requires tag values as arrays: `{"name": [tag_name]}` NOT `{"name": tag_name}`
This was a bug that caused 400 errors.

## sync_amocrm.py Architecture

### Functions
| Function | Description |
|----------|-------------|
| `sync_contacts()` | Sync amoCRM contacts → raw.amo_contacts |
| `sync_leads()` | Sync amoCRM leads → raw.amo_leads |
| `sync_events()` | Sync amoCRM events → raw.amo_events |
| `sync_notes()` | Sync amoCRM notes → raw.amo_notes |
| `run_amocrm_core_transformations()` | Raw→Core: dim_client, fact_deal |
| `main()` | Orchestrate all syncs |

### Important: client_id Mapping
- `fact_deal.client_id` = **NEGATIVE** of `amo_contact_id`
- Queries MUST use `ABS(fd.client_id)` or `-fd.client_id` to match dim_client
- Always filter `client_id > 0` to exclude walk-ins

## sync_amo_events.py
- Syncs amoCRM event log (status changes, calls, notes)
- ⚠️ Line 35: DB_URL must use real password (not `***`)
- Cron: runs via `sync_amo_events_cron.sh`

## sync_birthdays_to_amo.py
- Pushes birthday info from Altegio clients to amoCRM contacts
- Reads from PostgreSQL, writes to amoCRM API

## Cron Schedule
- Main sync (Altegio+amoCRM): `0 22 * * *` (daily_sync_with_notify.sh)
- Events sync: via `sync_amo_events_cron.sh`

## Running

```bash
# Full amoCRM sync
cd /root/.openclaw/workspace/openclaw-mcp-servers
.venv/bin/python3 scripts/sync_amocrm.py

# Events sync
.venv/bin/python3 scripts/sync_amo_events.py

# Birthday push
.venv/bin/python3 scripts/sync_birthdays_to_amo.py
```

## Troubleshooting

### API errors
1. Check `AMOCRM_ACCESS_TOKEN` in `.env` — tokens expire!
2. Check rate limits (429): AmoClient has `_429_retries` counter
3. `ensure_tag` 400 errors → ensure array format: `{"name": [tag_name]}`

### Data quality
```sql
-- Check contact sync
SELECT COUNT(*) FROM raw.amo_contacts;

-- Check lead sync
SELECT COUNT(*) FROM raw.amo_leads;

-- Verify client_id mapping
SELECT COUNT(*) FROM core.fact_deal WHERE client_id < 0;
```

## Pitfalls
- `ensure_tag` must use array format `{"name": [tag_name]}` (not string)
- `fact_deal.client_id` is NEGATIVE of amo_contact_id
- amoCRM API tokens expire and need refresh
- 429 retry in amo_client.py has no hard limit — potential infinite recursion
- `sync_amo_events.py:35` may have hardcoded `***` password — verify