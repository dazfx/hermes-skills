---
name: stale-leads-automation
version: 1.0.0
category: devops
description: amoCRM ЗАВИСШИЕ (stale leads) automation — auto-classifies and routes stuck leads to automated or manual processing pipelines.
triggers:
  - stale leads processing
  - зависшие leads
  - pipeline 9476838
  - amo_automation
---

# Stale Leads Automation (amoCRM ЗАВИСШИЕ)

## Overview
Daily automation that processes leads stuck in the "Зависшие" (stuck/stale) status of pipeline 9476838, routing them either to automated re-engagement (Зависшие АВТО) or manual handling (ЗАВИСШИЕ Ручная).

## Key Resources
- **Script**: `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/stale_leads_automation.py`
- **State file**: `scripts/stale_leads_state.json` (tracks processed leads, prevents reprocessing)
- **amo_automation toolkit**: `/root/.openclaw/workspace/openclaw-mcp-servers/amo_automation/`
- **Cron**: Runs daily at 06:45 UTC

## Pipeline Configuration
| Parameter | Value |
|---|---|
| Pipeline ID | 9476838 |
| Source Status | 82208374 (Зависшие) |
| Auto Target | 85358010 (Зависшие АВТО) |
| Manual Target | 85358014 (ЗАВИСШИЕ Ручная) |
| Tag | Зависшая |
| Batch Size | 20 leads per batch per target |

## How It Works
1. Fetches leads from status 82208374 (Зависшие) in pipeline 9476838
2. Checks state file to skip already-processed leads
3. Splits leads into two batches (auto vs manual) — up to 20 per target per run
4. Moves batch 1 → status 85358010 (automated re-engagement pipeline)
5. Moves batch 2 → status 85358014 (manual handling pipeline)
6. Tags all processed leads with "Зависшая"
7. Updates state file with newly processed lead IDs

## Manual Execution
```bash
cd /root/.openclaw/workspace/openclaw-mcp-servers
python3 scripts/stale_leads_automation.py
```

## Troubleshooting
- **No leads processed**: Check state file — all leads may already be tracked. Reset by clearing stale IDs from state file.
- **Script fails**: Verify amoCRM API token is valid. Check network connectivity.
- **Partial batch**: If < 20 leads available, only available leads are processed. This is normal.
- **State file corruption**: Back up `scripts/stale_leads_state.json` before editing. Format: JSON with array of processed lead IDs.

## Pitfalls
- Do NOT delete the state file unless you want to reprocess ALL leads (causes duplicate moves)
- The tag "Зависшая" must match exactly (case-sensitive, Cyrillic)
- Batch limits (20) are per-target — so up to 40 leads total per run
- If cron didn't fire, check syslog: `grep CRON /var/log/syslog | grep stale`