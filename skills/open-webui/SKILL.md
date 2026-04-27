---
name: open-webui
description: Install and configure Open WebUI with Docker, connect to Ollama, set up nginx reverse proxy
version: 1.0
---

# Open WebUI Installation

## Prerequisites
- Docker and Docker Compose installed
- Ollama running (systemd service or standalone)
- Nginx installed (for reverse proxy)

## Installation Steps

### 1. Create docker-compose.yml

Place in a project directory (e.g., /opt/open-webui/):

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always

volumes:
  open-webui:
    external: true
```

### 2. Connect Ollama

Ollama by default only listens on 127.0.0.1. Docker needs it accessible via the host gateway.

**Preferred fix:** Add `Environment="OLLAMA_HOST=0.0.0.0"` to the Ollama systemd service file, then daemon-reload and restart Ollama. This makes Ollama listen on all interfaces so Docker can reach it via host.docker.internal.

### 3. Launch

```bash
cd /opt/open-webui/ && docker compose up -d
```

**Pitfall:** `docker run` commands may be blocked by security scanners when they contain HTTP URL environment variables. Use `docker compose` instead.

### 4. Create Admin Account

Open WebUI's first registered user gets admin role automatically. You must register via the web UI at http://HOST:3000.

**Pitfall:** The API signup endpoints return "Method Not Allowed" -- you MUST use the web UI for initial admin creation.

### 5. Nginx Reverse Proxy

Configure nginx to proxy external traffic to Open WebUI. Use a separate listen port (e.g., 3001) proxying to localhost:3000. Key requirements for the proxy config:

- Standard proxy headers (Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto)
- `proxy_http_version 1.1` for WebSocket support
- `Upgrade` and `Connection` headers for WebSocket upgrade
- `proxy_read_timeout 86400` to prevent long-lived WebSocket connections from being killed

After creating the config, symlink it into sites-enabled, test with `nginx -t`, and reload.

### 6. Port Conflict Check

Open WebUI uses port 8080 internally. If port 8080 is already in use on the host, map to a different port (e.g., 3000:8080).

## Verification

```bash
docker ps | grep open-webui
curl -s http://localhost:3000/api/config
docker logs open-webui 2>&1 | tail -20
```

## Managing Models

### Setting Default Model
1. Log in, click model selector on main chat page, pick model, click "Set as default"

### Making Models Public (visible to all users)
Models are Private by default. For each model:
1. Admin Panel > Settings > Models > click model card > "Access" button
2. Change dropdown from "Private" to "Public"
3. Click "Save & Update"

### Ollama Cloud Models
Cloud models (suffix `:cloud`) appear automatically when Ollama API connection is active.

### API Notes for Automation
- API auth requires BOTH cookie AND Bearer token (use `-c`/`-b` curl flags)
- Endpoint `/api/v1/models` returns JSON; trailing slash `/api/v1/models/` returns HTML
- Setting `access_control: null` via API does NOT change Private to Public; use UI instead

## Ollama Access from Docker

Ollama binds to localhost by default. Docker containers need it reachable via host.docker.internal. Set the OLLAMA_HOST environment variable in the Ollama service config to make it listen on all interfaces.

## Pitfalls Summary

1. Port 8080 conflict -- Open WebUI uses 8080 internally; map to a different host port
2. Ollama localhost binding -- Set OLLAMA_HOST for Docker container access
3. Docker run blocked -- Security scanners may block docker run with HTTP URLs; use docker compose
4. API signup disabled -- Must use web UI for first admin account
5. WebSocket support -- nginx MUST include Upgrade/Connection headers and long proxy_read_timeout
6. Models are Private by default -- Must manually set each model to Public via UI
7. API auth requires cookie + Bearer token -- Bearer alone returns HTML
8. API trailing slash matters -- `/api/v1/models` = JSON, `/api/v1/models/` = HTML
9. Browser sessions expire quickly -- re-login needed between operations