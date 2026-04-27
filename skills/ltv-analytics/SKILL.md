---
name: ltv-analytics
description: AX2-compatible Lifetime Value calculation for the analytics database. Includes KW filters, diagnostic skip, transaction type mapping, and LTV formulas.
version: 1.0
---

# LTV Analytics Skill

AX2-compatible Lifetime Value calculation for the analytics database.

## Architecture

- **MCP Tool**: `get_ltv_analysis_tool(date_from, date_to, kw_filter, group_by)` in analytics-mcp
- **SQL File**: LTV query with KW filters, diagnostic skip, transaction breakdown
- **Materialized View**: `mart.client_ltv_real` (aggregate, no KW filter)

## Key Formulas

```
LTV  = firstCash + repeatCash + goodsRev + abonRev
LTV2 = LTV + balance
deposit_spent = GREATEST(0, deposit_topups - balance)
```

## KW Filters (AX2-compatible)

- **KW_INC**: `rf, ipl, carbon, glow, hidra, limpieza` — client must match at least ONE in (name + phone + first_service)
- **KW_EXC**: `харченко, clipper, eduard, едуард, test, 9001800, тест, клипер, катаева, kataeva, степкина, kharchencko, stepkina` — client must match NONE

## Diagnostic Skip Logic

First qualifying visit = first visit with amount > 0 that is NOT solely diagnostic. If ALL visits are diagnostic, the first one is used.

DIAG_KEYWORDS: `diagnóstico, diagnostico, диагностика, консультац, primera consulta, diagnostico facial`

## Transaction Type Mapping

| Altegio `sold_item_type` | `fact_client_account_transaction.type` | AX2 Name |
|---|---|---|
| `deposit_topup` | `deposit` | abonRev |
| `goods_transaction` | `retail` | goodsRev |
| `service` | `service_payment` | service revenue |

## Critical Data Points

- `fact_deal.client_id = NEGATIVE amo_contact_id` — use `ABS()` or `-int(amo_id)`
- `client_id > 0` in ALL queries (walk-in = -1, excluded)
- `attendance_status = 'attended'` for visits, `amount > 0` for revenue
- `cash_amount` = real revenue (NOT `amount`)
- Concatenated duplicate IDs auto-cleaned in sync

## Files

- `/root/.openclaw/workspace/openclaw-mcp-servers/analytics-mcp/tools/ltv.py` — MCP tool implementation
- `/root/.openclaw/workspace/openclaw-mcp-servers/analytics-mcp/server.py` — tool registration
- `/root/.openclaw/workspace/openclaw-mcp-servers/scripts/sync_altegio.py` — ETL with type mapping