---
name: daily-ops
version: 1.0.0
category: devops
description: Daily operations automation — sync pipelines, reports, crawlers, watchdogs, and model rotation for Alexey's server.
triggers:
  - daily operations
  - daily sync
  - daily report
  - automation watcher
  - task watchdog
  - alefmed crawler
  - model rotation
triggers_cron:
  - "04:00 UTC - crawl_alefmed.py"
  - "06:45 UTC - stale_leads_automation.py"
  - "08:00 UTC - daily_report.sh"
  - "11:55 UTC - rotate_crash_test_model.py"
  - "22:00 UTC - daily_sync_with_notify.sh"
  - "23:30 UTC - ax2_cron.php"
  - "*/15 - automation_watcher.sh"
  - "*/30 - task_watchdog.sh"
---

# Daily Operations Automation

## Overview
Suite of cron-driven scripts handling daily synchronization, reporting, crawling, service health monitoring, and model rotation.

## Script Inventory

### 1. Daily Sync with Notify (`daily_sync_with_notify.sh`)
- **Cron**: 22:00 UTC daily
- **Pipeline**: Altegio → amoCRM → LTV calculation → Snapshot → Segment generation → MCP restart → Telegram notification
- **Path**: Check crontab for exact location
- **Notifications**: Sends completion/status message via Telegram

### 2. Daily Report (`daily_report.sh`)
- **Cron**: 08:00 UTC daily
- **Log**: `/root/.openclaw/workspace/logs/daily_report.log`
- **Purpose**: Generates and delivers daily analytics report
- **Note**: Depends on data synced by the previous day's sync
- **Crontab entry**: `0 8 * * * /root/.openclaw/workspace/scripts/daily_report.sh >> /root/.openclaw/workspace/logs/daily_report.log 2>&1`

### 3. Alefmed Crawler (`crawl_alefmed.py`)
- **Cron**: 04:00 UTC daily
- **Purpose**: Website crawler for Alefmed content
- **Path**: Check scripts directory

### 4. Automation Watcher (`automation_watcher.sh`)
- **Cron**: Every 15 minutes
- **Purpose**: Service health watchdog — checks that automation services are running, restarts if down
- **Critical**: If this service itself dies, no auto-recovery for others

### 5. Task Watchdog (`task_watchdog.sh`)
- **Cron**: Every 30 minutes
- **Purpose**: Checks for overdue/stuck tasks in amoCRM, escalates or notifies

### 6. Model Rotation (`rotate_crash_test_model.py`)
- **Cron**: 11:55 UTC daily
- **Purpose**: Rotates the crash-test LLM model between glm-5.1 and kimi-k2.6
- **Behavior**: Alternates daily to distribute testing load

### 7. AX2 LTV Dashboard (`ax2_cron.php`)
- **Cron**: 23:30 UTC daily
- **Purpose**: Updates the AX2 LTV dashboard with fresh data
- **Stack**: PHP

## Dependency Chain
```
04:00  crawl_alefmed.py
06:45  stale_leads_automation.py  (separate skill)
08:00  daily_report.sh           (needs previous day's data)
11:55  rotate_crash_test_model.py
22:00  daily_sync_with_notify.sh (main sync pipeline)
23:30  ax2_cron.php              (LTV dashboard update)
```

Watchdog services run continuously:
- `automation_watcher.sh` → every 15 min
- `task_watchdog.sh` → every 30 min

## Manual Execution
```bash
# Check crontab for exact paths and run any script directly
crontab -l

# Example manual runs (adjust paths from crontab output):
# bash /path/to/daily_sync_with_notify.sh
# bash /path/to/daily_report.sh
# python3 /path/to/crawl_alefmed.py
# bash /path/to/automation_watcher.sh
# bash /path/to/task_watchdog.sh
# python3 /path/to/rotate_crash_test_model.py
```

## Troubleshooting
- **Sync failure at 22:00**: Check Altegio API connectivity and credentials. Verify Telegram bot token for notifications.
- **Report not delivered**: Confirm Telegram channel configuration. Check if sync from previous day succeeded.
- **Watcher not firing**: `grep CRON /var/log/syslog | tail -20`. Check script permissions (`chmod +x`).
- **Model rotation stuck**: Check which model is active: look at config or logs. Manually toggle if needed.
- **AX2 PHP failure**: Verify PHP CLI is available (`which php`). Check database connectivity.

## Pitfalls
- Daily report (08:00) depends on PREVIOUS day's sync (22:00) — if sync failed, report will be stale
- Model rotation only works for glm-5.1 ↔ kimi-k2.6 — adding a third model requires code changes
- automation_watcher.sh cannot recover itself if it dies — consider a separate systemd timer as backup
- All times are UTC — verify server timezone matches expectations
- **Cron job logging**: every cron job MUST redirect output to a log file (`>> /path/to/log 2>&1`). Jobs without log redirection silently lose their output — check `crontab -l` for entries missing `>>` and fix them.
- **Scripts source .env**: daily_report.sh and task_watchdog.sh need `source /root/.openclaw/workspace/openclaw-mcp-servers/.env` for TELEGRAM_BOT_TOKEN — if Telegram notifications silently fail, check the .env path in each script