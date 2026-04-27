# Hermes Agent Skills Hub

Custom skills for the Hermes Agent on Alexey's server (Ubuntu 24.04, 83.138.53.219).

## Skills

### DevOps
| Skill | Description |
|-------|-------------|
| `server-admin` | Server administration — health checks, services, nginx, Docker |
| `altegio-sync` | Altegio ETL sync — data mappings, known bugs, troubleshooting |
| `amocrm-automation` | amoCRM sync scripts + AmoClient — contacts, leads, events, birthdays |
| `automation-monitoring` | Service health + auto-repair watcher, task watchdog |
| `meta-ads-mcp` | Meta Ads API integration via MCP server |
| `open-webui` | Open WebUI installation and configuration |
| `stale-leads-automation` | amoCRM ЗАВИСШИЕ (stuck leads) automation |
| `daily-ops` | Daily operations — sync, reports, crawlers, watchdogs |
| `mcp-server-management` | MCP server stack lifecycle management |
| `filebrowser` | File Browser web file manager — install, configure, manage |
| `ltv-analytics` | AX2-compatible LTV calculation — KW filters, diagnostic skip |
| `planka` | Deploy Planka (open-source Trello-clone) via Docker |
| `webhook-subscriptions` | Webhook subscriptions for event-driven integrations |
| `streamlit-dashboard` | Streamlit analytics dashboard on :8501 |
| `openclaw-crash-test` | Systematic crash-testing for CRM/analytics |
| `openclaw-ecosystem` | ⚠️ DEPRECATED — migrated to Hermes |

### Data Science
| Skill | Description |
|-------|-------------|
| `analytics-database` | PostgreSQL analytics database — schema, sync, known issues |
| `jupyter-live-kernel` | Live Jupyter kernel for stateful Python exploration |

## Access Points
- **File Browser**: http://83.138.53.219:8090
- **Open WebUI**: http://83.138.53.219:3001
- **AX2 Dashboard**: http://83.138.53.219:8080
- **Streamlit**: http://83.138.53.219:8501

## Installation
```bash
# Skills auto-load from ~/.hermes/skills/ by Hermes Agent
# To add from this repo:
cp -r skills/<name> ~/.hermes/skills/devops/
```

## Auto-Sync Script
```bash
cd ~/.hermes/hermes-skills
for skill in skills/*/; do
    name=$(basename "$skill")
    for cat in devops data-science; do
        [ -d ~/.hermes/skills/$cat/$name ] && { rm -rf skills/$name; cp -r ~/.hermes/skills/$cat/$name skills/$name; }
    done
done
git add -A && git commit -m "sync: $(date +%Y-%m-%d)" && git push
```