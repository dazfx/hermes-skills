---
name: automation-monitoring
description: Service health monitoring and auto-repair — automation_watcher.sh + task_watchdog.sh. Checks MCP servers, PostgreSQL, Ollama, Streamlit, swap, and Telegram notifications.
version: 1.0
---

# Automation Monitoring Skill

## Overview
Two monitoring scripts that keep the system healthy: `automation_watcher.sh` (comprehensive health checker + auto-repair) and `task_watchdog.sh` (cron task freshness monitor).

## File Locations
- `automation_watcher.sh`: `~/.hermes/scripts/automation_watcher.sh`
- `task_watchdog.sh`: `~/.hermes/scripts/task_watchdog.sh`
- Logs: `/var/log/`
- Repairs log: `/var/log/repairs.log`

## Cron Schedule
- Watcher: `*/15 * * * *` (every 15 min)
- Task watchdog: `*/30 * * * *` (every 30 min)

## automation_watcher.sh

### What It Checks
| Service | Check | Auto-repair |
|---------|-------|-------------|
| PostgreSQL | pg_isready | `systemctl restart postgresql` |
| Analytics MCP | curl :8101/sse | `systemctl restart analytics-mcp` |
| Wazzup24 MCP | curl :8102/sse | `systemctl restart wazzup24-mcp` |
| Meta-Ads MCP | curl :8103/sse | `systemctl restart meta-ads-mcp` |
| Ollama | curl :11434/api/tags | `systemctl restart ollama` |
| Streamlit | curl :8501/_stcore/health | `systemctl restart streamlit-dashboard` |
| Swap usage | free -m | Warning if >80% |
| Sync freshness | /var/log/analytics_sync.log | Warning if >26h old |

### Auto-Repair Logic
- **5-minute cooldown**: Won't restart the same service twice within 5 minutes
- **10-second double-check**: Waits 10s after restart, verifies service is UP
- **Restart-loop detection**: Only acts if service is actually DOWN
- Uses HERMES bot token for Telegram notifications (not dead OpenClaw bot)
- **Dedup**: Won't send duplicate alerts within cooldown window

### Key Config
```bash
SCRIPTS_DIR="/root/.hermes/scripts"
CHAT_ID="215708742"  # Alexey
# Source Hermes .env for all tokens (NOT OpenClaw workspace!)
source /root/.hermes/.env
```

### Notification
Sends to Telegram via `@shostka_help_bot` (hermes bot token).
Includes dedup logic — won't spam about same issue repeatedly.

## task_watchdog.sh

### What It Checks
- Freshness of cron task logs (analytics_sync, daily_report, etc.)
- Flags tasks that haven't run in expected timeframe
- Reports stale/failed tasks

### Running
```bash
# Manual run
/root/.openclaw/workspace/scripts/automation_watcher.sh
/root/.openclaw/workspace/scripts/task_watchdog.sh
```

## Troubleshooting

### Watcher not firing
1. Check cron: `crontab -l | grep watcher`
2. Check log: `tail /root/.openclaw/workspace/logs/automation_watcher.log`
3. Verify bot token: `grep HERMES_BOT_TOKEN /root/.hermes/.env`

### Service keeps restarting
1. Check repairs log: `cat /root/.openclaw/workspace/logs/repairs.log`
2. Check service journal: `journalctl -u <service> --since '1 hour ago'`
3. Cooldown is 5 min — if restarts >5 min apart, watcher won't block them

### Telegram notifications not working
1. Bot token must be from HERMES `.env` (not OpenClaw)
2. Check `HERMES_BOT_TOKEN` extraction in script
3. Test: `curl -s "https://api.telegram.org/bot<TOKEN>/sendMessage?chat_id=215708742&text=test"`

## Pitfalls
- **CRITICAL: Watcher uses Hermes .env** — scripts source `/root/.hermes/.env`. If env vars like `POSTGRES_PASSWORD` or `AMOCRM_ACCESS_TOKEN` are missing, watcher can't connect to services and will falsely restart them every cycle (e.g. PostgreSQL restart loop every 15 min). Always verify `source /root/.hermes/.env` exports all needed vars.
- Watcher uses ANASASIA_BOT_TOKEN for notifications (with fallback to TELEGRAM_BOT_TOKEN)
- 5-minute cooldown prevents restart loops but won't help if service crashes every 6 min
- Swap warnings occur frequently if memory is tight — consider adding swap or reducing services
- FileBrowser at :8090 is NOT checked by watcher (add if needed)