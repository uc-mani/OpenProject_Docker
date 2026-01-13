# OpenProject LAN Deployment on Windows (Docker Compose, HTTP)

This guide shows how to deploy **OpenProject on a Windows machine** using **Docker Desktop + Docker Compose** so that **other team members on the same LAN** can access it over **plain HTTP** (HTTPS can be added later).

> Recommended LAN URL example: `http://openproject.angularspring.com`  
> (internal DNS record pointing to your Windows PC’s LAN IP)

---

## 1) What “LAN deployment” means vs “local”

- **Local deployment**: Only accessible on the same PC via `http://localhost:<port>`.
- **LAN deployment**: Accessible from **other PCs** on the network via:
  - `http://<LAN-IP>:<port>` or
  - `http://openproject.angularspring.com` (preferred, using internal DNS)

LAN access requires:
- Publishing a port on the Windows host (Compose port mapping)
- Allowing that port through Windows Firewall
- Stable LAN IP / internal DNS mapping

---

## 2) Prerequisites

### A) Windows & Hardware
- Windows 10/11 (64-bit)
- Virtualization enabled in BIOS/UEFI
- Recommended: 8–16 GB RAM, SSD storage

### B) Software
- **Docker Desktop** (Linux containers)
- **Git for Windows**
- Optional: a text editor (VS Code)

### C) Network
- Your Windows PC must have a **stable LAN IP**:
  - Preferred: router **DHCP reservation**
  - Alternative: static IP
- Decide your access pattern:
  - **No port in URL**: use host port **80**
  - **With port** (simpler): use host port **8080**

### D) DNS (recommended for professional org use)
Ask company IT/admin to create an internal DNS record:

- `openproject.angularspring.com` → `YOUR_WINDOWS_PC_LAN_IP`

If internal DNS is not available yet:
- Temporary option: add a hosts-file entry on each client machine

### E) SMTP (for invites & notifications)
To send invitations/notifications, OpenProject needs a working SMTP server.
Typically, you must get these from your company IT/admin:
- SMTP host (server address)
- SMTP port (often 587)
- Authentication: username/password or IP-based relay
- TLS settings: STARTTLS/SSL requirements
- Allowed sender address (e.g., `openproject@angularspring.com`)
- HELO/EHLO domain (use your domain, e.g., `angularspring.com`)

---

## 3) Recommended deployment source (official)

Use the official OpenProject Docker Compose setup:

- https://github.com/opf/openproject-docker-compose

---

## 4) Step-by-step deployment (HTTP, LAN)

### Step 1 — Choose an installation folder
Example:
- `D:\WORK\OpenProject`

Open **PowerShell** and run:

```powershell
cd D:\WORK
git clone -b stable/16 https://github.com/opf/openproject-docker-compose.git
cd openproject-docker-compose
```

> You can use a different stable branch if needed (follow OpenProject release notes).

---

### Step 2 — Create `.env`
Copy the sample environment file:

```powershell
Copy-Item .env.example .env
```

Open `.env` in your editor and set:

#### Required for LAN + HTTP
```env
# Preferred LAN hostname (use internal DNS if possible)
OPENPROJECT_HOST__NAME=openproject.angularspring.com

# Run OpenProject in HTTP mode (you can switch to HTTPS later)
OPENPROJECT_HTTPS=false

# Port exposed on the Windows host
# Use 80 for: http://openproject.angularspring.com
# Or use 8080 for: http://openproject.angularspring.com:8080
PORT=8080
```

#### Data persistence locations
Use project-relative folders (simple for Windows):

```env
PGDATA=./pgdata
OPDATA=./opdata
```

> **Important:** Your folder creation must match these paths.  
> If `OPDATA=./opdata`, create `./opdata` (not `./assets`).

---

### Step 3 — Create data folders (Windows)
Run this from the repository root:

```powershell
New-Item -ItemType Directory -Force -Path .\pgdata, .\opdata | Out-Null
```

---

### Step 4 — Pull images and start containers
```powershell
docker compose pull
docker compose up -d
```

Check status:
```powershell
docker compose ps
```

View logs:
```powershell
docker compose logs -f --tail=200
```

---

### Step 5 — Allow access through Windows Firewall
Create an inbound rule for your chosen port.

If using **PORT=80**:
```powershell
New-NetFirewallRule -DisplayName "OpenProject HTTP (80)" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
```

If using **PORT=8080**:
```powershell
New-NetFirewallRule -DisplayName "OpenProject HTTP (8080)" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

---

### Step 6 — Test access locally, then from another PC

Local test:
- If `PORT=8080`: `http://localhost:8080`
- If `PORT=80`: `http://localhost`

LAN test from another device:
- `http://<YOUR_WINDOWS_LAN_IP>:8080` (or `:80`)
- Or: `http://openproject.angularspring.com` (preferred, with internal DNS)

To find your LAN IP:
```powershell
ipconfig
```

---

### Step 7 — First-time setup in OpenProject
OpenProject will guide you through:
- Admin account setup
- Organization settings
- Basic configuration

After login, confirm:
- You can create users
- You can create a project and assign tasks
- Email settings (see next section) if you want invites/notifications

---

## 5) Configure SMTP for invites and notifications

### What you need from IT/admin
Ask for:
- SMTP host (e.g., `smtp.angularspring.com` or Microsoft 365 relay host)
- Port (commonly 587)
- Authentication method:
  - Username/password (preferred), OR
  - Relay allowed from the OpenProject server IP
- TLS/STARTTLS requirement
- Allowed sender address (e.g., `openproject@angularspring.com`)
- HELO/EHLO domain: `angularspring.com`

### Where to configure in OpenProject
In OpenProject UI:
- **Administration** → **Emails and notifications** → **SMTP settings**
- Enter host/port/authentication details
- Send a test email (if the UI provides test functionality)

> If SMTP is not configured correctly, invites may be created in the UI but **emails won’t be delivered**.

---

## 6) HTTPS later (recommended upgrade path)

For professional use, HTTPS is strongly recommended.

Typical options:
- **Reverse proxy** (Nginx / Caddy / Traefik) in front of OpenProject
- Use a certificate from:
  - Company internal CA (internal-only)
  - Let’s Encrypt (publicly reachable domain/ports)

When you switch to HTTPS, you must ensure OpenProject is aware it is served via HTTPS
(so links/cookies are correct). Follow OpenProject documentation for reverse proxy / HTTPS setups.

---

## 7) Operations: backups, updates, and reliability

### A) Backups (highly recommended)
Back up both:
- **Database** (`pgdata`)
- **OpenProject data** (`opdata`) — attachments, etc.

Consider a scheduled backup job to another disk or NAS.

### B) Updates
General safe workflow:
```powershell
docker compose pull
docker compose up -d
docker compose logs -f --tail=200
```

For major version upgrades, follow OpenProject upgrade notes and test in staging first.

### C) Reliability tips
- Put the PC on a UPS if possible
- Ensure Docker Desktop is configured to start on boot (if you want always-on service)
- Use DHCP reservation/static IP to avoid breaking DNS mapping

---

## 8) Troubleshooting

### Port already in use
If the selected port is taken (common for 80), change `PORT` in `.env` to `8080`.

### Containers running but site not reachable from LAN
Check:
- Port mapping (`docker compose ps`)
- Windows Firewall inbound rule exists for the port
- Your PC is on the same LAN/VLAN as the clients (or routing rules allow access)

### Check logs
```powershell
docker compose logs -f --tail=200
```

---

## 9) Helpful links

- Official OpenProject Docker Compose repository:  
  https://github.com/opf/openproject-docker-compose

- OpenProject documentation:  
  https://www.openproject.org/docs/

- Reference community guide (useful examples & notes):  
  https://github.com/uc-mani/OpenProject_Docker

---

## Appendix: Recommended DNS record

Create an internal DNS record:
- Name: `openproject.angularspring.com`
- Type: `A`
- Value: `YOUR_WINDOWS_PC_LAN_IP`

Then users access:
- `http://openproject.angularspring.com` (if PORT=80)
- `http://openproject.angularspring.com:8080` (if PORT=8080)
