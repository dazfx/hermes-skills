# Hermes Agent Skills Hub

Custom skills for the Hermes Agent on Alexey's server (Ubuntu 24.04, 83.138.53.219).

## Skills

### DevOps
| Skill | Description |
|-------|-------------|
| `server-admin` | Server administration — health checks, services, nginx, Docker, filebrowser |
| `openclaw-ecosystem` | OpenClaw MCP ecosystem audit and maintenance |
| `openclaw-crash-test` | Systematic crash-testing for OpenClaw CRM/analytics |
| `meta-ads-mcp` | Meta Ads API integration via MCP server |
| `open-webui` | Open WebUI installation and configuration |
| `stale-leads-automation` | amoCRM ЗАВИСШИЕ (stuck leads) automation |
| `daily-ops` | Daily operations — sync, reports, crawlers, watchdogs |
| `mcp-server-management` | MCP server stack lifecycle management |
| `filebrowser` | File Browser web file manager — install, configure, manage |
| `ltv-analytics` | AX2-compatible LTV calculation — KW filters, diagnostic skip, formulas |
| `planka` | Deploy Planka (open-source Trello-clone Kanban board) via Docker |
| `webhook-subscriptions` | Create and manage webhook subscriptions for event-driven integrations |

### Data Science
| Skill | Description |
|-------|-------------|
| `analytics-database` | PostgreSQL analytics database admin — schema, sync, known issues |
| `jupyter-live-kernel` | Live Jupyter kernel for stateful, iterative Python data exploration |

## Access Points
- **File Browser**: http://83.138.53.219:8090
- **Open WebUI**: http://83.138.53.219:3001
- **AX2 Dashboard**: http://83.138.53.219:8080

## Installation
```bash
# Skills are auto-loaded from ~/.hermes/skills/ by Hermes Agent
# To add a skill from this repo:
cp -r skills/<skill-name> ~/.hermes/skills/devops/
# or for data-science skills:
cp -r skills/<skill-name> ~/.hermes/skills/data-science/
```

## Auto-Sync Script
```bash
# Sync all custom skills to this repo (run from server)
cd ~/.hermes/hermes-skills
for skill in skills/*/; do
    name=$(basename "$skill")
    # Update from local if exists
    for cat in devops data-science; do
        if [ -d ~/.hermes/skills/$cat/$name ]; then
            rm -rf skills/$name
            cp -r ~/.hermes/skills/$cat/$name skills/$name
        fi
    done
done
git add -A && git commit -m "sync: $(date +%Y-%m-%d)" && git push
```