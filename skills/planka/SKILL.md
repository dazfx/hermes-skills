---
name: planka
description: Deploy Planka (open-source Trello-clone Kanban board) via Docker Compose, configure auth, and bootstrap projects/boards via API
version: 1.0
---

# Planka Deployment Skill

## Docker Compose Setup

Create `/opt/planka/docker-compose.yml`:

```yaml
version: '3.9'
services:
  planka-db:
    image: postgres:16-alpine
    container_name: planka-db
    restart: always
    environment:
      POSTGRES_USER: planka
      POSTGRES_PASSWORD: <PASSWORD>
      POSTGRES_DB: planka
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U planka"]
      interval: 10s
      timeout: 5s
      retries: 5

  planka:
    image: ghcr.io/plankanban/planka:latest
    container_name: planka
    restart: always
    depends_on:
      planka-db:
        condition: service_healthy
    environment:
      BASE_URL: http://<SERVER_IP>:1337
      DATABASE_URL: postgresql://planka:<PASSWORD>@planka-db:5432/planka
      DEFAULT_ADMIN_EMAIL: admin@example.com
      DEFAULT_ADMIN_PASSWORD: <PASSWORD>
      DEFAULT_ADMIN_NAME: Admin
      DEFAULT_ADMIN_USERNAME: admin
    ports:
      - "1337:1337"
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:1337/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  db-data:
```

Start: `cd /opt/planka && docker compose up -d`

## Terms Acceptance Flow (CRITICAL)

Planka requires terms acceptance before API login works. The flow is:

1. **Get terms signature**: `curl -s http://localhost:1337/api/terms` → `item.signature`
2. **Login** (returns pendingToken, not access token): `POST /api/access-tokens` with `emailOrUsername` + `password` → `pendingToken`
3. **Accept terms**: `POST /api/access-tokens/accept-terms` with `{"pendingToken": "<token>", "signature": "<signature>"}` → `item` (access token)

**IMPORTANT**: Terminal tool truncates long strings (JWT tokens ~225 chars). Always save tokens to files:

```bash
# Login and save token to file
curl -s http://localhost:1337/api/access-tokens \
  -X POST -H 'Content-Type: application/json' \
  -d '{"emailOrUsername":"admin@example.com","password":"<PASSWORD>"}' \
  -o /tmp/planka_login.json

# Extract pending token, accept terms, save access token
python3 -c "
import json
login = json.load(open('/tmp/planka_login.json'))
pt = login['pendingToken']
terms = json.load(open('/tmp/terms.json'))  # save /api/terms response here
sig = terms['item']['signature']
payload = json.dumps({'pendingToken': pt, 'signature': sig})
with open('/tmp/accept.json', 'w') as f:
    f.write(payload)
"

curl -s http://localhost:1337/api/access-tokens/accept-terms \
  -X POST -H 'Content-Type: application/json' -d @/tmp/accept.json \
  -o /tmp/auth_token.json
```

**Alternative**: Set `terms_signature` and `terms_accepted_at` directly in DB, then restart container:
```bash
docker exec planka-db psql -U planka -d planka -c "UPDATE user_account SET terms_signature='accepted', terms_accepted_at=NOW() WHERE email='admin@example.com';"
docker restart planka
```

After restart, login returns access token directly (no terms step needed).

## API Quirks

### Projects
- **Required fields**: `name`, `type` ("private" or "shared")
- Example: `POST /api/projects {"name": "My Project", "type": "private"}`

### Boards
- **Required fields**: `name`, `position`
- Board is created under a project via `POST /api/projects/{projectId}/boards`

### Lists (columns)
- **Required fields**: `name`, `type`, `position`
- `type` must be `"active"` or `"closed"` (NOT "task-list" or "kanban")
- `position` uses fibonacci-style spacing (65535, 131070, 196605...)
- Created via `POST /api/boards/{boardId}/lists`

### Cards
- **Required fields**: `name`, `position`
- Created via `POST /api/lists/{listId}/cards`

### Auth Header
All authenticated requests need: `Authorization: Bearer <token>`

## Nginx Proxy (optional)

If you want a subdomain:
```nginx
server {
    listen 80;
    server_name planka.example.com;
    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Then add SSL with certbot: `certbot --nginx -d planka.example.com`

## Pitfalls

1. **Token truncation**: Terminal tool truncates output at ~200 chars. JWT tokens are ~225 chars. Always use `-o /tmp/file.json` and extract with python.
2. **Terms of Service**: Can't skip it via env var (TERMS_OF_SERVICE_ENABLED doesn't work in latest Planka). Must accept via API or DB.
3. **Docker port conflict**: If port 5432 is in use by system Postgres, the container still works — it maps to a different internal port.
4. **Planka card `type` field**: Lists use `type: "active"`, NOT "task-list" or "kanban". Projects use `type: "private"` or `"shared"`.