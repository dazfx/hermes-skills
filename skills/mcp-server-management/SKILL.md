---
name: mcp-server-management
version: 1.0.0
category: devops
description: MCP server stack management — lifecycle control for analytics, wazzup24, meta-ads, amocrm, and context7 MCP servers.
triggers:
  - mcp server restart
  - mcp server health
  - restart analytics
  - restart mcp stack
  - full restart
  - port 8101
  - port 8102
  - port 8103
---

# MCP Server Stack Management

## Overview
Manages the fleet of MCP (Model Context Protocol) servers running on Alexey's infrastructure. Includes Python SSE servers and Node.js/npx servers spawned via Hermes config.

## Server Inventory

### Python SSE Servers (long-running processes)
| Server | Port | Protocol | Path |
|---|---|---|---|
| analytics-mcp | 8101 | SSE | `/root/.openclaw/workspace/openclaw-mcp-servers/analytics-mcp/` |
| wazzup24-mcp | 8102 | SSE | `/root/.openclaw/workspace/openclaw-mcp-servers/wazzup24-mcp/` |
| meta-ads-mcp | 8103 | SSE | `/root/.openclaw/workspace/openclaw-mcp-servers/meta-ads-mcp/` |

### Managed Servers (spawned by Hermes)
| Server | Type | Launch Method |
|---|---|---|
| amocrm-mcp | Node.js | Hermes config (auto-spawned) |
| context7 | npx | Hermes config (auto-spawned) |

## Restart Scripts
All located in `/root/.openclaw/workspace/openclaw-mcp-servers/`:

| Script | Scope |
|---|---|
| `full_restart.sh` | Restarts entire MCP stack + Hermes-managed servers |
| `restart_analytics.sh` | Restarts only analytics-mcp (port 8101) |
| `restart_mcp_stack.sh` | Restarts all Python SSE servers (8101–8103) |

## Common Operations

### Check if a server is running
```bash
# By port
ss -tlnp | grep 8101   # analytics
ss -tlnp | grep 8102   # wazzup24
ss -tlnp | grep 8103   # meta-ads

# By process
ps aux | grep analytics-mcp
ps aux | grep wazzup24-mcp
ps aux | grep meta-ads-mcp
```

### Restart individual server
```bash
cd /root/.openclaw/workspace/openclaw-mcp-servers

# Analytics only
bash restart_analytics.sh

# All Python SSE servers
bash restart_mcp_stack.sh

# Full stack (Python SSE + Hermes-managed)
bash full_restart.sh
```

### Check server health (SSE endpoints)
```bash
curl -s http://localhost:8101/sse  &  # analytics
curl -s http://localhost:8102/sse  &  # wazzup24
curl -s http://localhost:8103/sse  &  # meta-ads
```

### View logs
```bash
# Check for log files in each server directory
ls /root/.openclaw/workspace/openclaw-mcp-servers/analytics-mcp/*.log
ls /root/.openclaw/workspace/openclaw-mcp-servers/wazzup24-mcp/*.log
ls /root/.openclaw/workspace/openclaw-mcp-servers/meta-ads-mcp/*.log

# Or check systemd/journal if services are registered
journalctl -u analytics-mcp --since "1 hour ago" -n 50
```

## Troubleshooting

### Port already in use
```bash
# Find process holding the port
lsof -i :8101
# Kill it
kill -9 <PID>
# Then restart
bash restart_analytics.sh
```

### Server starts but SSE not responding
- Check firewall: `ufw status` — port must be open for internal access
- Check the SSE endpoint path — may differ per server
- Verify Python dependencies: `pip install -r requirements.txt` in server dir

### Hermes-managed servers (amocrm-mcp, context7) not connecting
- These are auto-spawned — check Hermes agent config (`~/.hermes/config.yaml`)
- Restart Hermes agent itself if needed
- Check Node.js version: `node --version` (amocrm-mcp requires Node.js)

### Full stack not recovering
```bash
# Nuclear option — kill all and restart
bash full_restart.sh

# If that fails, check for zombie processes
ps aux | grep -E '(analytics|wazzup|meta-ads)-mcp' | grep -v grep
# Kill zombies, then restart
```

## Pitfalls
- SSE servers are long-running — a crashed process won't auto-restart unless `automation_watcher.sh` catches it (see daily-ops skill)
- Port conflicts: If a restart script doesn't kill the old process first, the new one will fail to bind
- analytics-mcp (8101) is depended on by daily sync — if it's down during 22:00 UTC sync, data pipeline breaks
- Hermes-managed servers (amocrm-mcp, context7) are NOT controlled by restart_mcp_stack.sh — use full_restart.sh or restart Hermes
- Always check port availability before starting: `ss -tlnp | grep 81`