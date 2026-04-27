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

### Data Science
| Skill | Description |
|-------|-------------|
| `analytics-database` | PostgreSQL analytics database admin — schema, sync, known issues |

## Access Points
- **File Browser**: http://83.138.53.219:8090
- **Open WebUI**: http://83.138.53.219:3001
- **AX2 Dashboard**: http://83.138.53.219:8080

## Installation
```bash
# Add as external skills directory in ~/.hermes/config.yaml
skills:
  external_dirs:
    - /root/.hermes/skills
```

