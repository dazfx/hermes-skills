---
name: daily-ops
version: 2.0.0
category: devops
description: Daily operations automation — sync pipelines, reports, crawlers, watchdogs, and model rotation for Alexey's server.
---

# Daily Operations Automation (Hermes)

All scripts: `/root/.hermes/scripts/`. All source `/root/.hermes/.env`.
MCP workspace: `/root/.hermes/mcp-servers/` (venv at `venv/bin/python3`).

## Cron Schedule (system crontab)

| Time (UTC) | Job | Script |
|-----------|-----|--------|
| 22:00 | Daily sync + notify | `~/.hermes/scripts/daily_sync_with_notify.sh` |
| 23:30 | AX2 cache update | `/var/www/ax/ax2_cron.php` |
| 04:00 | Website crawler | `~/.hermes/mcp-servers/venv/bin/python3 ~/.hermes/mcp-servers/scripts/crawl_alefmed.py` |
| 06:45 | ЗАВИСШИЕ (PAUSED) | `stale_leads_automation.py` |
| */30 | Task watchdog | `~/.hermes/scripts/task_watchdog.sh` |
| 08:00 | Daily report | `~/.hermes/scripts/daily_report.sh` |
| 11:55 | Model rotation | `~/.hermes/mcp-servers/venv/bin/python3 ~/.hermes/scripts/rotate_crash_test_model.py` |
| */15 | Automation watcher | `~/.hermes/scripts/automation_watcher.sh` |

## Script Inventory

### automation_watcher.sh
Service health watchdog — checks analytics-mcp, wazzup24-mcp, meta-ads-mcp, streamlit-dashboard, PostgreSQL. Auto-restarts on failure. Dedup-based alerting to @anastasialeadbot.

### task_watchdog.sh
Checks MCP service status, sync freshness (≤26h), disk space. Alerts via @anastasialeadbot only on new problems.

### daily_sync_with_notify.sh
Pipeline: Altegio → amoCRM → LTV refresh → Snapshot → Segments → MCP restart. Reports via @anastasialeadbot.

### daily_report.sh
Morning health snapshot: services, DB, disk, RAM, load. Sends to @anastasialeadbot.

## Pitfalls
- All scripts source `/root/.hermes/.env` (not OpenClaw workspace!)
- `BOT_TOKEN="${ANASTASIA_BOT_TOKEN:-$TELEGRAM_BOT_TOKEN}"` in all notification scripts
- `WORKSPACE="/root/.hermes/mcp-servers"` and `PYTHON="$WORKSPACE/venv/bin/python3"` in automation_watcher + daily_sync
- ZAVISSHIE (stale leads) is PAUSED — no auto-classification
- Daily report at 08:00 depends on PREVIOUS day's sync at 22:00