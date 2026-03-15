# matrix-setup

A complete guide to self-hosting a **private** Matrix (Synapse) chat server for a friend group — with voice/video calling, end-to-end encryption, and native app support.

## Stack

| Component | Role |
|-----------|------|
| [Synapse](https://github.com/element-hq/synapse) | Matrix homeserver |
| PostgreSQL 16 | Database |
| [LiveKit](https://livekit.io) | Voice/video SFU (media routing) |
| [lk-jwt-service](https://github.com/element-hq/lk-jwt-service) | Bridges Matrix auth to LiveKit |
| [Element Web](https://element.io) | Browser client |
| [Caddy](https://caddyserver.com) | Reverse proxy + automatic HTTPS |

All services run as Docker containers managed with Docker Compose.

## Design

- **Federation disabled** — fully closed, private server
- **Token-based registration** — admin generates invite codes; friends create their own accounts
- **Voice/video** — group calls via LiveKit SFU (Element Call / MatrixRTC)
- **End-to-end encryption** — enabled by default in all Matrix clients

## Target hardware

Hostinger KVM 2 (2 vCPU / 8GB RAM / 100GB NVMe) running Ubuntu 24.04. Should work on any comparable VPS.

## DNS

Two subdomains required:

```
chat.yourdomain.com      →  main server + Element Web client
livekit.yourdomain.com   →  voice/video media server
```

## Guide

See [matrix-setup-redacted.md](matrix-setup-redacted.md) for the full step-by-step setup guide covering:

1. VPS purchase and initial hardening
2. Docker installation
3. Secret generation
4. Configuration files (`docker-compose.yml`, `Caddyfile`, `homeserver.yaml`, `livekit.yaml`, `element-config.json`)
5. Synapse setup and tuning
6. Launching and verifying all services
7. Creating the admin account and generating invite tokens for friends
8. Connecting clients (Element Web, Element Desktop, Cinny, Element X mobile)
9. Setting up Spaces and rooms
10. Automated database backups
11. Database maintenance
12. Ongoing maintenance and updates
13. Migrating to a new VPS

## Supported clients

- **Browser**: Element Web (served directly from your domain)
- **Desktop**: Element Desktop, Cinny
- **Mobile**: Element X (iOS / Android)

## Quick start (after setup)

```bash
# Start everything
cd ~/matrix && docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f

# Update containers
docker compose pull && docker compose up -d && docker image prune -f
```
