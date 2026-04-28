---
name: hermes-ecosystem
description: Hermes Agent ecosystem reference — all services, cron jobs, MCP servers, bots, and access points for Alexey's server
version: 1.0
category: devops
---

# Hermes Ecosystem Overview

## Core Services
| Service | Port | Status |
|---------|------|--------|
| Hermes Gateway | — | ✅ Active |
| Hermes Agent (Telegram) | @shostka_help_bot | ✅ |
| PostgreSQL analytics | 5432 | ✅ |
| Ollama (cloud) | 11434 | ✅ |
| Open WebUI | 3001 | ✅ |
| Streamlit Dashboard | 8501 | ✅ |
| File Browser | 8090 | ✅ |
| AX2 Dashboard (PHP) | 8080 | ✅ |
| Nginx | 80/443 | ✅ |

## MCP Servers (systemd)
- `mcp-analytics-server` — PostgreSQL analytics queries
- `mcp-wazzup24-server` — Wazzup24 chat API
- `mcp-meta-ads-server` — Meta Ads API (4 accounts)
- `mcp-amocrm-server` — amoCRM (stdio transport)
- `mcp-context7-server` — Documentation lookup

## Cron Jobs (system crontab)
| Time (UTC) | Job | Script |
|-----------|-----|--------|
| 22:00 | Daily sync + notify | `~/.hermes/scripts/daily_sync_with_notify.sh` |
| 23:30 | AX2 cache update | `/var/www/ax/ax2_cron.php` |
| 04:00 | Website crawler | `~/.hermes/scripts/crawl_alefmed.py` |
| 06:45 | ЗАВИСШИЕ (PAUSED) | `stale_leads_automation.py` |
| */30 | Task watchdog | `~/.hermes/scripts/task_watchdog.sh` |
| 08:00 | Daily report | `~/.hermes/scripts/daily_report.sh` |
| 11:55 | Model rotation | `~/.hermes/scripts/rotate_crash_test_model.py` |
| */15 | Automation watcher | `~/.hermes/scripts/automation_watcher.sh` |

## Memory Architecture
- **USER.md** — User profile (who Alexey is)
- **MEMORY.md** — Durable facts, preferences, conventions
- **session_search** — Cross-session recall via SQLite FTS5
- **Compression** — `compressor` engine, threshold=0.5, target_ratio=0.2

## Access Points
- File Browser: http://83.138.53.219:8090 (admin / AlefMed2026!@#)
- Open WebUI: http://83.138.53.219:3001
- AX2 Dashboard: http://83.138.53.219:8080
- Streamlit: http://83.138.53.219:8501

## OpenClaw → Hermes Migration
OpenClaw gateway + claude-mem are STOPPED and DISABLED since Apr 27.
All cron jobs now reference `~/.hermes/` paths (symlinks to OpenClaw workspace).
Bot @anastasialeadbot is dead. Use @shostka_help_bot.
