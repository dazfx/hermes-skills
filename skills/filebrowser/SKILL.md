---
name: filebrowser
description: Install and configure File Browser web file manager with systemd and nginx reverse proxy for external access.
version: 1.0
---

# File Browser — Web File Manager Setup

## Installation
```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/filebrowser/master/get.sh | bash
# Binary installed to /usr/local/bin/filebrowser
```

## Configuration

### 1. Create systemd service
```ini
# /etc/systemd/system/filebrowser.service
[Unit]
Description=File Browser
After=network.target

[Service]
ExecStart=/usr/local/bin/filebrowser -r / --port 8089 --address 127.0.0.1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- `-r /` = root directory (serve entire filesystem)
- `--port 8089` = internal port (localhost only)
- `--address 127.0.0.1` = bind localhost (nginx proxies externally)

### 2. Create nginx reverse proxy
```nginx
# /etc/nginx/sites-available/filebrowser
server {
    listen 8090;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8089;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket + SSE support (File Browser uses WS for real-time updates)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/filebrowser /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx
```

### 3. Enable and start
```bash
systemctl daemon-reload
systemctl enable filebrowser
systemctl start filebrowser
```

### 4. Open firewall
```bash
ufw allow 8090/tcp
```

### 5. Default credentials
- URL: `http://<SERVER_IP>:8090`
- Default: admin / admin (change on first login!)

## Current Deployment (Alexey's Server)
- Server: 83.138.53.219
- Port: 8090 (nginx proxy) → 8089 (filebrowser)
- Root: `/` (full filesystem)
- Credentials: admin / AlefMed2026!@#
- Version: v2.63.2

## Pitfalls
- **Bind to 127.0.0.1 only** — never expose filebrowser directly; always use nginx proxy for external access
- **Nginx must have WebSocket support** — File Browser uses WS for real-time file changes; without upgrade headers, UI hangs on file operations
- **UFW must allow the nginx port (8090)**, not the internal port (8089)
- **Change default password immediately** — admin/admin is the default and is publicly known