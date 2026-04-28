---
name: openclaw-hermes-migration
description: | 
  Полная миграция сервисов и cron-задач из экосистемы OpenClaw в автономный Hermes Agent.
  Порядок действий: копирование кода MCP-серверов, обновление systemd-юнитов, 
  чистка crontab, отключение OpenClaw gateway. Используется при переходе 
  от OpenClaw к Hermes или восстановлении после сбоя.
version: 1.0.0
author: Hermes Agent
tags: [migration, devops, openclaw, hermes, systemd]
---

# Миграция OpenClaw → Hermes

## Когда использовать

- Переход с OpenClaw на автономный Hermes
- Дублирование MCP-серверов (запущены из OpenClaw workspace)
- Очистка crontab от путей OpenClaw
- Отключение OpenClaw gateway

## Шаг 1: Копирование кода MCP-серверов

```bash
# Создать директорию и venv
mkdir -p ~/.hermes/mcp-servers
python3 -m venv ~/.hermes/mcp-servers/venv

# Скопировать серверы
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/analytics-mcp/ ~/.hermes/mcp-servers/analytics-mcp/
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/wazzup24-mcp/ ~/.hermes/mcp-servers/wazzup24-mcp/
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/meta-ads-mcp/ ~/.hermes/mcp-servers/meta-ads-mcp/
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/amocrm-mcp-patched/ ~/.hermes/mcp-servers/amocrm-mcp/
rsync -av /root/.openclaw/workspace/dashboard/ ~/.hermes/mcp-servers/dashboard/
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/scripts/ ~/.hermes/mcp-servers/scripts/
rsync -av /root/.openclaw/workspace/openclaw-mcp-servers/amo_automation/ ~/.hermes/mcp-servers/amo_automation/

# Установить зависимости
~/.hermes/mcp-servers/venv/bin/pip install -r ~/.hermes/mcp-servers/analytics-mcp/requirements.txt
~/.hermes/mcp-servers/venv/bin/pip install -r ~/.hermes/mcp-servers/wazzup24-mcp/requirements.txt
~/.hermes/mcp-servers/venv/bin/pip install streamlit
```

## Шаг 2: Обновление systemd-юнитов

Заменить все пути `/root/.openclaw/workspace/openclaw-mcp-servers/` → `/root/.hermes/mcp-servers/`

Шаблоны юнитов:

```
# analytics-mcp
[Unit]
Description=Analytics MCP Server (SSE)
After=network.target postgresql.service
[Service]
Type=simple; User=root
WorkingDirectory=/root/.hermes/mcp-servers/analytics-mcp
ExecStart=/root/.hermes/mcp-servers/venv/bin/python3 server.py --sse
Restart=always; RestartSec=10

# wazzup24-mcp
ExecStart=/root/.hermes/mcp-servers/venv/bin/python3 server.py --sse

# meta-ads-mcp (свой venv!)
ExecStart=/root/.hermes/mcp-servers/meta-ads-mcp/venv/bin/python server.py

# streamlit-dashboard
WorkingDirectory=/root/.hermes/mcp-servers/dashboard
ExecStart=/root/.hermes/mcp-servers/venv/bin/streamlit run app.py --server.port 8501 --server.address 0.0.0.0 --server.headless true --browser.gatherUsageStats false
```

После обновления:
```bash
systemctl daemon-reload
systemctl restart analytics-mcp wazzup24-mcp meta-ads-mcp streamlit-dashboard
```

## Шаг 3: Очистка crontab

```bash
crontab -l | sed 's|/root/.openclaw/workspace/openclaw-mcp-servers/.venv/bin/python3|/root/.hermes/mcp-servers/venv/bin/python3|g' | crontab -
```

Проверить: `crontab -l | grep openclaw` — должно быть пусто.

## Шаг 4: Обновление config.yaml (amoCRM)

```yaml
mcp_servers:
  amocrm:
    args:
    - /root/.hermes/mcp-servers/amocrm-mcp/dist/index.js  # было .../openclaw/.../amocrm-mcp-patched/
```

## Шаг 5: Отключение OpenClaw

```bash
systemctl stop openclaw-gateway
systemctl disable openclaw-gateway
systemctl mask openclaw-gateway
pkill -f openclaw  # убить оставшиеся процессы
```

## Шаг 6: Настройка бота уведомлений (если был отдельный бот в OpenClaw)

Если в OpenClaw был второй бот (например @anastasialeadbot) для уведомлений,
перенести его токен в Hermes и обновить скрипты.

### 6a. Извлечь токен из конфига OpenClaw

```bash
# Токен лежит в openclaw.json → channels.telegram.botToken
python3 -c "
import json
with open('/root/.openclaw/openclaw.json') as f:
    c = json.load(f)
print(c['channels']['telegram']['botToken'])
" > /tmp/anastasia_token.txt
```

### 6b. Добавить токен в Hermes .env

```bash
ANASTASIA_TOKEN=$(cat /tmp/anastasia_token.txt)
sed -i "/^TELEGRAM_BOT_TOKEN=529102/a ANASTASIA_BOT_TOKEN=${ANASTASIA_TOKEN}" /root/.hermes/.env
```

### 6c. Проверить бот

```bash
source /root/.hermes/.env
curl -s "https://api.telegram.org/bot${ANASTASIA_BOT_TOKEN}/getMe"
# Должен вернуть "ok": true
```

### 6d. Обновить скрипты уведомлений

Все скрипты должны:
- source `/root/.hermes/.env` (не OpenClaw workspace!)
- Использовать `ANASTASIA_BOT_TOKEN` с fallback: `BOT_TOKEN="${ANASTASIA_BOT_TOKEN:-$TELEGRAM_BOT_TOKEN}"`
- curl через `$BOT_TOKEN`

Скрипты для обновления:
- `automation_watcher.sh` — проверка сервисов + авто-ремонт
- `task_watchdog.sh` — статус MCP + sync + диск
- `daily_report.sh` — ежедневная сводка здоровья
- `daily_sync_with_notify.sh` — результат синхронизации

Шаблон патча для header скрипта:
```bash
# Source .env for bot tokens
set -a
source /root/.hermes/.env
set +a

BOT_TOKEN="${ANASTASIA_BOT_TOKEN:-$TELEGRAM_BOT_TOKEN}"
CHAT_ID="215708742"
```
И заменить `${TELEGRAM_BOT_TOKEN}` → `${BOT_TOKEN}` во всех curl-запросах.

## Шаг 7: Проверка

```bash
ps aux | grep openclaw  # должно быть пусто
systemctl list-units --type=service | grep openclaw  # пусто
ss -tlnp | grep 18789  # порт Gateway свободен

# Проверить что скрипты берут токен из Hermes
grep -l "source.*hermes.*\\.env" /root/.hermes/scripts/*.sh
grep "ANASTASIA_BOT_TOKEN" /root/.hermes/.env
```

### ⚠️ CRITICAL: Verify no watcher restart loops
After migration, the watcher may falsely restart services because it lost access to env vars.
```bash
# Verify watcher is healthy (not restarting services):
tail -50 /var/log/repairs.log
# Should have NO recent entries. If PostgreSQL/MCP keeps restarting every 15 min,
# check that ALL of these are in /root/.hermes/.env:
#   POSTGRES_PASSWORD, AMOCRM_ACCESS_TOKEN, ALTEGIO_API_TOKEN
grep -E '^(POSTGRES_PASSWORD|AMOCRM_ACCESS_TOKEN|ALTEGIO_API_TOKEN)=' /root/.hermes/.env
```
Root cause: watcher sources `.env` but OpenClaw's `.env` had vars that weren't copied to Hermes `.env`.
PostgreSQL `pg_isready` or `psql` fail without `POSTGRES_PASSWORD`, watcher interprets this as "PostgreSQL down" → restart loop.

## Порты (Hermes)

| Порт | Сервис |
|------|--------|
| 8101 | analytics-mcp |
| 8102 | wazzup24-mcp |
| 8103 | meta-ads-mcp |
| 8501 | Streamlit dashboard |
| 8090 | FileBrowser (nginx → :8089) |
| 3001 | Open WebUI |
