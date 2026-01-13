# OpenProject – Generic LAN Setup & Deployment (Docker Compose)

Deploy **OpenProject** on a server in your **local network (LAN)** so **multiple users** can access it (tasks, assignments, notifications) from their own machines.

This is a **generic** guide (not company-specific). It includes both a **simple LAN pilot** setup and a **recommended “professional”** setup with HTTPS via reverse proxy.

---

## Table of contents

- [1) LAN vs Localhost](#1-lan-vs-localhost)
- [2) Deployment options](#2-deployment-options)
- [3) Prerequisites](#3-prerequisites)
- [4) Network planning](#4-network-planning)
- [5) Install Docker on the server](#5-install-docker-on-the-server)
- [6) Deploy OpenProject with the official Compose stack](#6-deploy-openproject-with-the-official-compose-stack)
- [7) Option A: Direct LAN access (HTTP)](#7-option-a-direct-lan-access-http)
- [8) Option B: Reverse proxy + HTTPS (recommended)](#8-option-b-reverse-proxy--https-recommended)
- [9) Configure SMTP for email/invites](#9-configure-smtp-for-emailinvites)
- [10) Backups & restore](#10-backups--restore)
- [11) Upgrades](#11-upgrades)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Security hardening checklist](#13-security-hardening-checklist)
- [14) Helpful links](#14-helpful-links)

---

## 1) LAN vs Localhost

- **Localhost-only** binds to `127.0.0.1` → reachable **only** from the server itself.
- **LAN deployment** binds to `0.0.0.0` or a LAN IP (e.g., `192.168.1.20`) → reachable from **other devices** on the network.

If you want teammates to use OpenProject, you need **LAN deployment** (plus DNS/firewall).

---

## 2) Deployment options

### Option A — Direct LAN access (fastest)
- URL looks like: `http://openproject.company.lan:8080`
- Easiest for pilots inside a trusted network
- Still recommended to add HTTPS later

### Option B — Reverse proxy + HTTPS (recommended for professional use)
- URL looks like: `https://openproject.company.com`
- OpenProject binds only to `127.0.0.1:8080`
- Reverse proxy publishes `80/443` and handles TLS (Caddy/Nginx/Traefik/etc.)
- Best user experience + best security baseline

---

## 3) Prerequisites

### Server sizing (starting point)
- **CPU:** 2–4 vCPU
- **RAM:** 4–8 GB (more for large usage)
- **Disk:** SSD recommended; plan for uploads + DB growth + backups

### Software
- Linux server recommended (Ubuntu/Debian/RHEL-family)
- Docker Engine + Docker Compose plugin
- Git

### Organizational prerequisites
- A **static IP** or DHCP reservation for the server
- A **hostname** (DNS record preferred)
- SMTP relay credentials (from IT) for invites/notifications

---

## 4) Network planning

### Pick a hostname
Recommended approach: create an internal DNS A-record such as:

- `openproject.company.lan` → `192.168.1.20`

Temporary testing alternative: add a hosts-file entry on client machines.

- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Linux/macOS: `/etc/hosts`

Example hosts entry:

```
192.168.1.20  openproject.company.lan
```

### Decide your exposure model
- Option A (direct HTTP): open `8080/tcp` to the LAN
- Option B (reverse proxy): open `80/443` to the LAN; keep `8080` local-only

### Firewall planning
On the server, allow inbound ports you actually use:
- Option A: `8080/tcp`
- Option B: `80/tcp` and `443/tcp`

(Also check network firewalls/VLAN ACLs.)

---

## 5) Install Docker on the server

Install Docker Engine + Docker Compose from official Docker docs for your OS.

Verify:

```bash
docker version
docker compose version
```

If you want to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
# log out & log back in
```

---

## 6) Deploy OpenProject with the official Compose stack

OpenProject provides an official Docker Compose setup intended for self-hosting/production.

### 6.1 Create a deployment directory

```bash
sudo mkdir -p /opt/openproject
sudo chown -R $USER:$USER /opt/openproject
cd /opt/openproject
```

### 6.2 Clone the official repository

```bash
git clone https://github.com/opf/openproject-docker-compose.git
cd openproject-docker-compose
```

> Tip: The repo is versioned by stable branches (e.g., `stable/16`). For controlled upgrades, deploy from a stable branch that matches the major version you want.

### 6.3 Create your `.env` file

```bash
cp .env.example .env
```

**Important:** keep `.env` private (do not commit it).

---

## 7) Option A: Direct LAN access (HTTP)

Use this if you want the simplest possible LAN rollout first.

### 7.1 Edit `.env`

Set the public hostname and protocol:

```env
OPENPROJECT_HOST__NAME=openproject.company.lan:8080
OPENPROJECT_HTTPS=false
```

Expose OpenProject on the LAN:

```env
PORT=0.0.0.0:8080
```

### 7.2 Start

```bash
docker compose pull
docker compose up -d
docker compose logs -f
```

### 7.3 Test from another machine
Open:

- `http://openproject.company.lan:8080`

If unreachable, jump to [Troubleshooting](#12-troubleshooting).

---

## 8) Option B: Reverse proxy + HTTPS (recommended)

Use this for professional deployments (better URLs + encryption + cleaner networking).

### 8.1 Bind OpenProject locally only

In `.env`:

```env
OPENPROJECT_HOST__NAME=openproject.company.com
OPENPROJECT_HTTPS=true
PORT=127.0.0.1:8080
```

### 8.2 Put a reverse proxy in front

You can run the reverse proxy:
- on the same server (simplest), or
- on an existing gateway/ingress in your network

#### Example: Caddy (simple TLS)
Caddy can automatically request/renew Let’s Encrypt certs if the hostname is publicly valid and reachable.
If your hostname is internal-only, use an internal CA or self-signed certs.

A minimal Caddyfile concept:

```
openproject.company.com {
  reverse_proxy 127.0.0.1:8080
}
```

#### Example: Nginx (concept)
If Nginx terminates TLS, make sure it forwards the right headers:

```nginx
location / {
  proxy_pass http://127.0.0.1:8080;
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Host $host;
}
```

### 8.3 Start OpenProject (same as before)

```bash
docker compose pull
docker compose up -d
docker compose logs -f
```

Then access:

- `https://openproject.company.com`

---

## 9) Configure SMTP for email/invites

OpenProject uses SMTP to send:
- invitations
- password resets
- notifications/reminders

In most organizations, you need SMTP details from IT (or a known relay such as Microsoft 365 / Google Workspace / Exchange).

### What you typically need
- SMTP server hostname
- port (commonly `587` for STARTTLS or `465` for implicit TLS)
- username/password (service account or relay credentials)
- encryption method (STARTTLS vs SSL)
- HELO/EHLO domain (usually your company domain)

### Where to configure
- OpenProject UI: **Administration → Email notifications** (wording varies by version)
- Or via environment variables (recommended for repeatable deployments)

After configuring, send a test email and check OpenProject logs for SMTP errors.

---

## 10) Backups & restore

Back up at least:
1) PostgreSQL database  
2) Uploads/attachments volume (assets/opdata)  
3) Your `.env` + reverse-proxy config (store secrets safely)

### 10.1 Database backup (pg_dump)

```bash
docker compose exec -T db pg_dump -U postgres openproject > openproject-backup.pgdump
```

Restore:

```bash
cat openproject-backup.pgdump | docker compose exec -T db psql -U postgres openproject
```

### 10.2 Backup uploads/attachments

If attachments are stored in a Docker volume, you can archive them with a temporary container:

```bash
docker volume ls
# identify the assets/opdata volume name, then:
docker run --rm   -v <VOLUME_NAME>:/volume   -v $(pwd):/backup   alpine   sh -c "cd /volume && tar -czf /backup/openproject-assets.tar.gz ."
```

Store backups off-host (NAS/S3/backup server). Periodically test restore.

---

## 11) Upgrades

Recommended approach:
1. Backup first
2. Pull updated images
3. Restart services
4. Watch logs and verify UI

Typical commands:

```bash
docker compose pull
docker compose up -d
docker compose logs -f
```

If you deploy from a stable branch:
```bash
git pull
# optionally switch stable branches after reading release notes
```

---

## 12) Troubleshooting

### 12.1 “Works on server, not from another PC”
- Check `PORT` binding:
  - `PORT=127.0.0.1:8080` = localhost-only
  - `PORT=0.0.0.0:8080` = LAN-accessible
- Check OS firewall and network firewall/VLAN ACLs
- Verify DNS/hosts mapping is correct
- Confirm the server is reachable from that subnet

### 12.2 Email not sending
- Validate SMTP settings (server/port/credentials/encryption)
- Ensure containers can resolve DNS and reach SMTP server
- Check logs:
  ```bash
  docker compose logs --since 30m web
  docker compose logs --since 30m worker
  ```

---

## 13) Security hardening checklist

- Prefer **HTTPS** (reverse proxy)
- Restrict access to trusted networks (firewall/VPN/SSO)
- Keep OpenProject/Docker images updated
- Use strong admin credentials; disable unused accounts
- Disable self-registration if not needed
- Regular backups + restore tests
- Monitor disk usage (uploads + DB growth)

---

## 14) Helpful links

OpenProject:
- OpenProject Docker installation overview: https://www.openproject.org/docs/installation-and-operations/installation/docker/
- OpenProject Docker Compose stack (official): https://github.com/opf/openproject-docker-compose
- OpenProject docs (index): https://www.openproject.org/docs/
- OpenProject release notes: https://www.openproject.org/docs/release-notes/

Docker:
- Install Docker Engine: https://docs.docker.com/engine/install/
- Docker Compose: https://docs.docker.com/compose/
- Docker volumes: https://docs.docker.com/storage/volumes/
- Backup/restore volumes: https://docs.docker.com/storage/volumes/#backup-restore-or-migrate-data-volumes

Reverse proxy / TLS:
- Let’s Encrypt: https://letsencrypt.org/docs/
- Caddy: https://caddyserver.com/docs/
- Traefik: https://doc.traefik.io/traefik/
- Nginx: https://nginx.org/en/docs/

Reference examples (community):
- uc-mani/OpenProject_Docker: https://github.com/uc-mani/OpenProject_Docker
