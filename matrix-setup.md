# Self-Hosting Matrix (Synapse) on Hostinger VPS

A complete guide to running your own **private** Synapse chat server (using the Matrix protocol) voice/video
calling, end-to-end encryption, and native app support for a friend group.

**Stack**: Synapse + PostgreSQL + LiveKit + lk-jwt-service + Caddy + Element Web

**VPS**: Hostinger KVM 2 (2 vCPU, 8GB RAM, 100GB NVMe)

**Privacy posture**: Federation disabled (fully closed server),
token-based registration (admin generates invite codes, friends create their own accounts).

---

## Phase 1 — Buy and configure the VPS

### 1.1 Purchase

- Go to https://www.hostinger.com/vps-hosting
- Select the **KVM 2** plan (2 vCPU / 8GB RAM / 100GB NVMe)
- Choose **Ubuntu 24.04** as the OS
- Pick a **US central or east coast** server location (splits latency for bicoastal friends)
- Complete checkout (email + payment, no ID required)

### 1.2 Register a domain

```
Type: A    Host: chat       Answer: YOUR_VPS_IP    TTL: 300
Type: A    Host: livekit    Answer: YOUR_VPS_IP    TTL: 300
```

This gives you:
- `chat.yourdomain.com` — the main server + web client
- `livekit.yourdomain.com` — voice/video media server

This guide uses `chat.example.com` as a placeholder — replace it everywhere.

### 1.3 First SSH connection

From your local PC terminal:

```bash
ssh root@YOUR_VPS_IP
```

### 1.4 Secure the server

```bash
# Update everything
apt update && apt upgrade -y

# Create a non-root user
adduser matrix
usermod -aG sudo matrix

# Set up SSH key auth (run this FROM YOUR LOCAL MACHINE, not the server)
# ssh-copy-id matrix@YOUR_VPS_IP

# Back on the server — disable password auth
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart ssh

# Set up firewall
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 7881/tcp           # LiveKit TCP fallback
ufw allow 50000:50100/udp    # LiveKit media (voice/video)
ufw default deny incoming
ufw enable
```

Port 8448 (Matrix federation) is intentionally NOT opened. This is a private server.

From now on, SSH in as your `matrix` user:

```bash
ssh matrix@YOUR_VPS_IP
```

---

## Phase 2 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Log out and back in for group membership
exit
ssh matrix@YOUR_VPS_IP

# Verify
docker run hello-world
```

---

## Phase 3 — Set up the project directory and generate secrets

```bash
mkdir -p ~/matrix && cd ~/matrix

# Generate secrets — you'll need these in multiple config files
echo "DB_PASSWORD: $(openssl rand -hex 32)" > secrets.txt
echo "LIVEKIT_SECRET: $(openssl rand -hex 32)" >> secrets.txt
cat secrets.txt
```

Write these down or keep the file open. You'll paste them into configs below.

Then delete secrets.txt after you finish setting up:

```bash
rm ~/matrix/secrets.txt
```

---

## Phase 4 — Create configuration files

### 4.1 docker-compose.yml

Create `~/matrix/docker-compose.yml`:

```yaml
services:
  # --- Reverse proxy (auto HTTPS) ---
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - synapse
      - element

  # --- Matrix homeserver ---
  synapse:
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    volumes:
      - ./synapse-data:/data
    environment:
      - SYNAPSE_CONFIG_PATH=/data/homeserver.yaml
    depends_on:
      postgres:
        condition: service_healthy

  # --- Database ---
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: PASTE_DB_PASSWORD_HERE
      POSTGRES_DB: synapse
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse"]
      interval: 5s
      timeout: 5s
      retries: 5

  # --- Element Web (browser client) ---
  element:
    image: vectorim/element-web:latest
    restart: unless-stopped
    volumes:
      - ./element-config.json:/app/config.json

  # --- LiveKit SFU (voice/video media routing) ---
  livekit:
    image: livekit/livekit-server:latest
    restart: unless-stopped
    ports:
      - "7881:7881"
      - "50000-50100:50000-50100/udp"
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml
    command: --config /etc/livekit.yaml

  # --- LiveKit JWT service (bridges Matrix auth to LiveKit) ---
  lk-jwt-service:
    image: ghcr.io/element-hq/lk-jwt-service:latest
    restart: unless-stopped
    extra_hosts:
      - "chat.example.com:CADDY_IP_HERE"
    environment:
      - LIVEKIT_JWT_BIND=:8080
      - LIVEKIT_URL=wss://livekit.chat.example.com
      - LIVEKIT_KEY=devkey
      - LIVEKIT_SECRET=PASTE_LIVEKIT_SECRET_HERE
      - LIVEKIT_FULL_ACCESS_HOMESERVERS=chat.example.com

volumes:
  caddy_data:
  caddy_config:
  postgres_data:
```

**Replace before proceeding:**
- `PASTE_DB_PASSWORD_HERE` → DB_PASSWORD from secrets.txt
- `PASTE_LIVEKIT_SECRET_HERE` → LIVEKIT_SECRET from secrets.txt
- All instances of `chat.example.com` → your actual domain

### 4.2 Caddyfile

Create `~/matrix/Caddyfile`:

```
chat.example.com {
    # Well-known endpoints for Matrix client discovery + LiveKit
    handle /.well-known/matrix/client {
        header Content-Type application/json
        header Access-Control-Allow-Origin *
        respond `{"m.homeserver":{"base_url":"https://chat.example.com"},"org.matrix.msc4143.rtc_foci":[{"type":"livekit","livekit_service_url":"https://livekit.chat.example.com"}]}`
    }

    # Matrix client API
    reverse_proxy /_matrix/* synapse:8008
    reverse_proxy /_synapse/client/* synapse:8008
    reverse_proxy /_synapse/admin/* synapse:8008

    # Default: serve Element Web
    reverse_proxy /* element:80
}

# LiveKit subdomain — handles both JWT auth and SFU WebSocket
livekit.chat.example.com {
    # JWT auth service (clients request call tokens here)
    handle /sfu/get {
        reverse_proxy lk-jwt-service:8080
    }
    handle /healthz {
        reverse_proxy lk-jwt-service:8080
    }

    # LiveKit SFU (WebSocket connections for media)
    handle {
        reverse_proxy livekit:7880
    }    
}

chat.example.com:8448 {
    reverse_proxy synapse:8448
    }
```

Replace all instances of `chat.example.com` with your actual domain.

### 4.3 Element Web config

Create `~/matrix/element-config.json`:

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://chat.example.com",
            "server_name": "chat.example.com"
        }
    },
    "brand": "Checkmate",
    "default_theme": "dark",
    "room_directory": {
        "servers": ["chat.example.com"]
    },
    "features": {
        "feature_group_calls": true
    },
    "show_labs_settings": false
}
```

Replace all instances of `chat.example.com` with your actual domain.

### 4.4 LiveKit config

Create `~/matrix/livekit.yaml`:

```yaml
port: 7880
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 50100
  use_external_ip: true
room:
  auto_create: false
keys:
  devkey: PASTE_LIVEKIT_SECRET_HERE
logging:
  level: info
```

Replace `PASTE_LIVEKIT_SECRET_HERE` with LIVEKIT_SECRET from secrets.txt.

**Critical**: The `devkey` here and the `LIVEKIT_KEY` + `LIVEKIT_SECRET` in
docker-compose.yml must match exactly. `devkey` is the key name, and the
value after it is the secret.

---

## Phase 5 — Generate and configure Synapse

### 5.1 Generate initial config

```bash
cd ~/matrix

docker run -it --rm \
  -v $(pwd)/synapse-data:/data \
  -e SYNAPSE_SERVER_NAME=chat.example.com \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

### 5.2 Edit homeserver.yaml

```bash
cd ~/matrix
nano synapse-data/homeserver.yaml
```

Find and update/add ALL of the following sections:

**Database** — replace the entire sqlite section:

```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: PASTE_DB_PASSWORD_HERE
    database: synapse
    host: postgres
    cp_min: 5
    cp_max: 10
```

**Listeners**:

```yaml
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client]
        compress: false
  - port: 8090
    tls: false
    type: http
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [openid]
        compress: false
  - port: 8448
    tls: false
    type: http
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [federation]
        compress: false
```

**Registration** — allow signup with invite tokens only:

```yaml
enable_registration: true
registration_requires_token: true
```

**Federation** — disable completely:

```yaml
federation_domain_whitelist: []
```

**Element Call / MatrixRTC features** — required for voice/video:

```yaml
experimental_features:
  msc3266_enabled: true
  msc4222_enabled: true

max_event_delay_duration: 24h

rc_message:
  per_second: 0.5
  burst_count: 30

rc_delayed_event_mgmt:
  per_second: 1
  burst_count: 20
```

**Presence** — To reduce idle resource usage (optional), set to false:

```yaml
presence:
  enabled: true
```

**Media storage** — Adjustable based on how much storage is available:

```yaml
max_upload_size: 100M
```
**Additional**:

```yaml
serve_server_wellknown: true
```
Tells Synapse to serve its own /.well-known/matrix/server discovery endpoint, so services can find the homeserver without external DNS lookups.

```
```
**Remove or comment out trusted_key_servers** (not needed without federation):

```yaml
# trusted_key_servers:
#   - server_name: "matrix.org"
```

Save and exit (Ctrl+X, Y, Enter).

---

## Phase 6 — Launch everything

### 6.1 Start services

```bash
cd ~/matrix

# Start all services
docker compose up -d

# Watch the logs for errors (Ctrl+C to stop watching)
docker compose logs -f
```

### 6.2 Verify

Give it 30-60 seconds to settle, then:

```bash
# Check Synapse is responding
curl https://chat.example.com/_matrix/client/versions

# Check LiveKit JWT service health
curl https://livekit.chat.example.com/healthz

# Check all containers are running
docker compose ps
```

### 6.3 Troubleshooting first-run issues

| Problem | Fix |
|---------|-----|
| Caddy can't get HTTPS cert | DNS hasn't propagated. Wait 5-30 min, then `docker compose restart caddy` |
| Synapse can't connect to postgres | Password in homeserver.yaml doesn't match docker-compose.yml |
| 502 Bad Gateway | Synapse still starting. Wait a minute. |
| lk-jwt-service unhealthy | Check LIVEKIT_KEY/SECRET match between livekit.yaml and docker-compose.yml |
| Voice calls fail silently | Check that ports 7881/tcp and 50000-50100/udp are open in firewall |

Check individual logs:

```bash
docker compose logs synapse
docker compose logs postgres
docker compose logs caddy
docker compose logs livekit
docker compose logs lk-jwt-service
```

---

## Phase 7 — Create your admin account and invite friends

### 7.1 Create your admin account (one time only)

```bash
cd ~/matrix

docker compose exec synapse register_new_matrix_user \
  http://localhost:8008 \
  -c /data/homeserver.yaml \
  -u matrix \
  -p ['INSERT_MATRIX_LOGIN_PASSWORD_HERE'] \
  --admin
```

### 7.2 Get your admin access token

Log in to Element Web at `https://chat.example.com`. Then:

**Element Web**: Click your avatar → All Settings → Help & About →
scroll to "Access token" (click to reveal). Copy it.

### 7.3 Generate invite tokens for friends

```bash
# Single-use token (one friend)
curl -sS -X POST "https://chat.example.com/_synapse/admin/v1/registration_tokens/new" \
  -H "Authorization: Bearer ['ACCESS_TOKEN']" \
  -H "Content-Type: application/json" \
  -d '{"uses_allowed": 1}'
```

Returns:

```json
{
    "token": "abc123xyz",
    "uses_allowed": 1,
    "pending": 0,
    "completed": 0,
    "expiry_time": null
}
```

Send the `token` value to your friend.

**Bulk invite** (one code for multiple people):

```bash
curl -sS -X POST "https://chat.example.com/_synapse/admin/v1/registration_tokens/new" \
  -H "Authorization: Bearer ['ACCESS_TOKEN']" \
  -H "Content-Type: application/json" \
  -d '{"uses_allowed": 10}' 
```

**Token with 7-day expiry**:

```bash
EXPIRY=$(python3 -c "import time; print(int((time.time() + 7*86400) * 1000))")
curl -sS -X POST "https://chat.example.com/_synapse/admin/v1/registration_tokens/new" \
  -H "Authorization: Bearer ['ACCESS_TOKEN']" \
  -H "Content-Type: application/json" \
  -d "{\"uses_allowed\": 10, \"expiry_time\": $EXPIRY}"
```

**List all tokens**:

```bash
curl -sS "https://chat.example.com/_synapse/admin/v1/registration_tokens" \
  -H "Authorization: Bearer ['ACCESS_TOKEN']"
```

### 7.4 How friends sign up

1. Go to `https://chat.example.com`
2. Click **"Create account"**
3. Enter the registration token you gave them
4. Set value of Homeserver to chat.example.com
5. Choose their own username and password
6. Once they create username, you mustt send invitation to your new Matrix space

---

## Phase 8 — Connect clients

### Browser
Go to `https://chat.example.com` — Element Web loads. Log in.

### Desktop — Element Desktop
- Download from https://element.io/download (Linux: AppImage, .deb, or Flatpak)
- At login, click "Edit" next to server URL → enter `https://chat.example.com`

### Desktop — Cinny (more Discord-like UI)
- Download from https://cinny.in or Flatpak
- Set homeserver to `https://chat.example.com`
- Supports voice/video rooms as of March 2026

### Mobile — Element X (iOS / Android)
- Install from App Store or Google Play
- Tap "Sign in" → change server to `https://chat.example.com`

All clients sync. Messages, rooms, and call history are the same everywhere.

---

## Phase 9 — Set up Spaces and rooms

Once logged in as admin:

1. Click **+** next to Spaces → "Create a space"
2. Name it (your group name), set to **Private**
3. Create rooms inside the Space:
   - `#general` — main chat
   - `#music` — sharing music
   - `#voice` — voice/video hangouts
   - `#off-topic` — whatever
4. Invite friends to the Space — they get access to all rooms

### Voice/video calls

In any room, click the call button in the room header. Everyone in the room
gets notified and can join. Camera on = video call, camera off = voice only.
Works in Element Web, Element Desktop, Element X mobile, and Cinny.

### Custom emoji

In Cinny, you can import custom emoji packs. Once set up, everyone on the server can use them.

---

## Phase 10 — Backups

### Automated database backups

Create `~/matrix/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR=~/matrix/backups
mkdir -p $BACKUP_DIR

docker compose -f ~/matrix/docker-compose.yml exec -T postgres \
  pg_dump -U synapse synapse | \
  gzip > "$BACKUP_DIR/synapse-$(date +%Y%m%d-%H%M%S).sql.gz"

find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "$(date): Backup complete" >> "$BACKUP_DIR/backup.log"
```

Schedule it:

```bash
chmod +x ~/matrix/backup.sh
crontab -e
# Add:
0 3 * * * /home/matrix/matrix/backup.sh 2>&1
```

This runs at 3am UTC every day.

### Pull backups to your local machine

On your local machine run:

```bash
crontab -e
0 8 * * * rsync -avz matrix@YOUR_VPS_IP:~/matrix/backups/ ~/matrix-backups/
```

---

## Phase 11 — Database maintenance

Additional maintenance:

### Monthly PostgreSQL vacuum

```bash
docker compose exec postgres psql -U synapse -d synapse -c "VACUUM ANALYZE;"
```

Or cron it:

```bash
# First Sunday of each month at 4am
0 4 1-7 * 0 docker compose -f ~/matrix/docker-compose.yml exec -T postgres psql -U synapse -d synapse -c "VACUUM ANALYZE;"
```

### Check database size

```bash
docker compose exec postgres psql -U synapse -d synapse \
  -c "SELECT pg_size_pretty(pg_database_size('synapse'));"
```

### Check disk usage

```bash
df -h
docker system df
```

---

## Phase 12 — Ongoing maintenance

### Update containers (every few weeks)

```bash
cd ~/matrix
docker compose pull
docker compose up -d
docker image prune -f
```

### Monitor

```bash
htop                     # RAM/CPU
df -h                    # disk
docker compose ps        # container status
docker stats             # per-container resources
```

### Restart

```bash
docker compose restart            # everything
docker compose restart synapse    # just Synapse
docker compose logs -f synapse    # watch after restart
```

---

## Quick reference

| What | Where / Command |
|------|-----------------|
| Web client | `https://chat.example.com` |
| Matrix API | `https://chat.example.com/_matrix/` |
| LiveKit health | `https://livekit.chat.example.com/healthz` |
| Synapse config | `~/matrix/synapse-data/homeserver.yaml` |
| Docker Compose | `~/matrix/docker-compose.yml` |
| Caddy config | `~/matrix/Caddyfile` |
| LiveKit config | `~/matrix/livekit.yaml` |
| Element config | `~/matrix/element-config.json` |
| Secrets | `~/matrix/secrets.txt` (delete after setup) |
| Backups | `~/matrix/backups/` |
| View logs | `docker compose logs <service>` |
| Invite friends | See Phase 7 |
| Check DB size | `docker compose exec postgres psql -U synapse -d synapse -c "SELECT pg_size_pretty(pg_database_size('synapse'));"` |

---

## Migration to another VPS

1. Spin up new VPS, install Docker
2. Copy config: `rsync -avz ~/matrix/ matrix@NEW_IP:~/matrix/`
3. Dump DB: `docker compose exec -T postgres pg_dump -U synapse synapse > dump.sql`
4. Transfer: `scp dump.sql matrix@NEW_IP:~/matrix/`
5. On new server: `docker compose up -d postgres` (wait for healthy)
6. Restore: `cat dump.sql | docker compose exec -T postgres psql -U synapse synapse`
7. Start everything: `docker compose up -d`
8. Update DNS A records (both `chat` and `livekit`) to new IP
9. Wait for propagation
10. Done — friends notice nothing

---
# Fix voice calls after restart

If voice/video calls stop working after a `docker compose down && docker compose up -d`:

## Step 1 — Check Caddy's current IP

```bash
cd ~/matrix
docker compose exec caddy hostname -i
```

## Step 2 — Update docker-compose.yml

```bash
nano ~/matrix/docker-compose.yml
```

Find the `lk-jwt-service` section and update the IP in `extra_hosts`:

```yaml
    extra_hosts:
      - "chat.example.com:NEW_CADDY_IP_HERE"
```

## Step 3 — Restart the JWT service

```bash
docker compose up -d lk-jwt-service
```

Voice calls should work again.

