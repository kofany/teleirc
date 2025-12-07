# MediaShare Web Deployment Guide

This guide helps Claude (or a human) configure MediaShare with Caddy reverse proxy and systemd on a Linux server.

## Prerequisites

- Linux server (Debian/Ubuntu recommended)
- Domain pointing to server (e.g., `media.example.com`)
- Go 1.21+ installed (for building)
- Root/sudo access

## Quick Setup Checklist

1. [ ] Build MediaShare binary
2. [ ] Create system user and directories
3. [ ] Configure environment file
4. [ ] Create systemd service
5. [ ] Install and configure Caddy
6. [ ] Start services
7. [ ] Configure TeleIRC integration

---

## Step 1: Build MediaShare

```bash
# On build machine (or target if Go is installed)
cd /path/to/teleirc
go build -o mediashare ./cmd/mediashare

# Copy to target server
scp mediashare user@server:/tmp/
```

## Step 2: Create User and Directories

```bash
# Create system user
sudo useradd -r -s /bin/false mediashare

# Create directories
sudo mkdir -p /opt/mediashare/uploads
sudo mkdir -p /var/lib/mediashare

# Move binary
sudo mv /tmp/mediashare /opt/mediashare/
sudo chmod +x /opt/mediashare/mediashare

# Set permissions
sudo chown -R mediashare:mediashare /opt/mediashare
sudo chown -R mediashare:mediashare /var/lib/mediashare
```

## Step 3: Environment Configuration

Create `/opt/mediashare/.env`:

```bash
sudo tee /opt/mediashare/.env << 'EOF'
# MediaShare Configuration
MEDIASHARE_PORT=8080
MEDIASHARE_BASE_URL=https://media.example.com
MEDIASHARE_API_KEY=GENERATE_SECURE_KEY_HERE
MEDIASHARE_STORAGE_PATH=/opt/mediashare/uploads
MEDIASHARE_DB_PATH=/var/lib/mediashare/mediashare.db
MEDIASHARE_MAX_FILE_SIZE=52428800
MEDIASHARE_RETENTION_HOURS=72
MEDIASHARE_SERVICE_NAME=MediaShare
MEDIASHARE_LANGUAGE=pl
EOF

# Secure the file
sudo chmod 600 /opt/mediashare/.env
sudo chown mediashare:mediashare /opt/mediashare/.env
```

**Generate API key:**
```bash
openssl rand -base64 32
```

## Step 4: Systemd Service

Create `/etc/systemd/system/mediashare.service`:

```bash
sudo tee /etc/systemd/system/mediashare.service << 'EOF'
[Unit]
Description=MediaShare - Media hosting service for TeleIRC
After=network.target

[Service]
Type=simple
User=mediashare
Group=mediashare
WorkingDirectory=/opt/mediashare
EnvironmentFile=/opt/mediashare/.env
ExecStart=/opt/mediashare/mediashare
Restart=always
RestartSec=5

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/opt/mediashare/uploads /var/lib/mediashare
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
EOF

# Reload and enable
sudo systemctl daemon-reload
sudo systemctl enable mediashare
```

## Step 5: Caddy Configuration

### Install Caddy (if not installed)

```bash
# Debian/Ubuntu
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### Configure Caddy

Add to `/etc/caddy/Caddyfile`:

```caddyfile
media.example.com {
    # Reverse proxy to MediaShare
    reverse_proxy localhost:8080

    # Optional: limit upload size (50MB)
    request_body {
        max_size 50MB
    }

    # Logging
    log {
        output file /var/log/caddy/mediashare.log
        format json
    }

    # Security headers
    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
    }
}
```

### Restart Caddy

```bash
sudo systemctl restart caddy
```

## Step 6: Start Services

```bash
# Start MediaShare
sudo systemctl start mediashare

# Check status
sudo systemctl status mediashare

# View logs
sudo journalctl -u mediashare -f
```

## Step 7: TeleIRC Integration

Add to your TeleIRC `.env` file:

```bash
# MediaShare Integration
MEDIASHARE_ENABLED=true
MEDIASHARE_ENDPOINT=http://localhost:8080
MEDIASHARE_API_KEY=SAME_KEY_AS_MEDIASHARE
```

If TeleIRC runs on a different server:
```bash
MEDIASHARE_ENDPOINT=https://media.example.com
```

Restart TeleIRC after configuration changes.

---

## Verification

### Test upload (from server):

```bash
curl -X POST \
  -H "X-API-Key: YOUR_API_KEY" \
  -F "file=@test.jpg" \
  -F "username=testuser" \
  http://localhost:8080/upload
```

### Expected response:

```json
{
  "success": true,
  "url": "https://media.example.com/Ab3xK",
  "raw_url": "https://media.example.com/r/Ab3xK",
  "id": "Ab3xK",
  "filename": "test.jpg"
}
```

### Test landing page:

Open `https://media.example.com/Ab3xK` in browser.

---

## Troubleshooting

### Service won't start

```bash
# Check logs
sudo journalctl -u mediashare -n 50

# Common issues:
# - Permission denied: check directory ownership
# - Port in use: change MEDIASHARE_PORT
# - Database locked: check if another instance is running
```

### Caddy SSL issues

```bash
# Check Caddy logs
sudo journalctl -u caddy -n 50

# Ensure domain DNS is correct
dig media.example.com

# Test Caddy config
caddy validate --config /etc/caddy/Caddyfile
```

### Files not uploading

```bash
# Check disk space
df -h /opt/mediashare/uploads

# Check permissions
ls -la /opt/mediashare/uploads

# Test API key
curl -v -H "X-API-Key: YOUR_KEY" http://localhost:8080/health
```

### Cleanup not working

```bash
# Check database
sqlite3 /var/lib/mediashare/mediashare.db "SELECT id, filename, last_opened_at FROM files ORDER BY last_opened_at;"

# Manual cleanup trigger (restart service)
sudo systemctl restart mediashare
```

---

## Nginx Alternative

If using Nginx instead of Caddy:

```nginx
server {
    listen 443 ssl http2;
    server_name media.example.com;

    ssl_certificate /etc/letsencrypt/live/media.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/media.example.com/privkey.pem;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name media.example.com;
    return 301 https://$server_name$request_uri;
}
```

---

## Security Recommendations

1. **API Key**: Use a strong, randomly generated key (32+ characters)
2. **Firewall**: Only expose port 443, keep 8080 internal
3. **Rate limiting**: Add rate limits in Caddy/Nginx for `/upload`
4. **Monitoring**: Set up alerts for disk space and service health
5. **Backups**: Backup `/var/lib/mediashare/mediashare.db` regularly

---

## Configuration Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MEDIASHARE_PORT` | No | 8080 | HTTP server port |
| `MEDIASHARE_BASE_URL` | **Yes** | - | Public URL (with https://) |
| `MEDIASHARE_API_KEY` | **Yes** | - | Authentication key |
| `MEDIASHARE_STORAGE_PATH` | No | ./uploads | File storage directory |
| `MEDIASHARE_DB_PATH` | No | ./mediashare.db | SQLite database path |
| `MEDIASHARE_MAX_FILE_SIZE` | No | 52428800 | Max upload size (bytes) |
| `MEDIASHARE_RETENTION_HOURS` | No | 72 | Auto-delete threshold |
| `MEDIASHARE_SERVICE_NAME` | No | MediaShare | HTML branding |
| `MEDIASHARE_LANGUAGE` | No | pl | UI language (pl/en) |
