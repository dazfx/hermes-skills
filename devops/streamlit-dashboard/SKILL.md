---
name: streamlit-dashboard
description: Streamlit analytics dashboard on :8501 — manage the service, deploy updates, troubleshoot display issues.
version: 1.0
---

# Streamlit Dashboard Skill

## Overview
Streamlit-based analytics dashboard running on port 8501 for data visualization.

## Service
- **Unit**: `streamlit-dashboard.service`
- **Port**: 8501 (exposed on 0.0.0.0)
- **Working Dir**: `/root/.openclaw/workspace/dashboard`
- **App**: `app.py`
- **Venv**: `/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/streamlit`
- **Depends on**: `postgresql.service`

## Service Management

```bash
# Status
systemctl status streamlit-dashboard

# Restart
systemctl restart streamlit-dashboard

# Logs
journalctl -u streamlit-dashboard --since '1 hour ago'

# Health check
curl -s http://localhost:8501/_stcore/health
```

## Configuration
```ini
# /etc/systemd/system/streamlit-dashboard.service
ExecStart=... streamlit run app.py \
  --server.port 8501 \
  --server.address 0.0.0.0 \
  --server.headless true \
  --browser.gatherUsageStats false
Restart=always
RestartSec=10
```

## Access
- URL: `http://83.138.53.219:8501`
- No authentication by default (Streamlit doesn't natively support it)

## Nginx Proxy (optional)
Not currently proxied via nginx. Direct access on :8501.

## Troubleshooting

### Dashboard not loading
1. Check service: `systemctl status streamlit-dashboard`
2. Check health: `curl http://localhost:8501/_stcore/health`
3. Check logs: `journalctl -u streamlit-dashboard -n 50`
4. Restart: `systemctl restart streamlit-dashboard`

### Data not showing
1. PostgreSQL must be running: `systemctl status postgresql`
2. Check if mart views are populated: `SELECT COUNT(*) FROM mart.client_ltv_real`
3. Materialized views may need refresh: `REFRESH MATERIALIZED VIEW mart.client_ltv_real`

### Port conflicts
1. Check what's on 8501: `ss -tlnp | grep 8501`
2. Kill stale processes if needed

## Pitfalls
- Streamlit caches data aggressively — may need `?rerun=true` parameter or restart
- No auth — anyone with the IP can view dashboard
- Depends on PostgreSQL + materialized views being fresh
- Uses same venv as MCP servers