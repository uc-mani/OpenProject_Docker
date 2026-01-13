# OpenProject (Localhost) — Docker (Compose) Setup

This guide sets up **OpenProject locally on your machine** using Docker Compose, accessible at **http://localhost:8080**.

> ✅ “Localhost setup” means the app binds to `127.0.0.1` so it’s reachable **only from the same computer**.

---

## Prerequisites

Install: (follow the instructions on the links below)

- **Docker Desktop** (Windows/macOS) or **Docker Engine** (Linux) (https://docs.docker.com/get-started/get-docker/)
- **Docker Compose v2** (usually included as `docker compose`) (https://www.openproject.org/docs/installation-and-operations/installation/docker-compose/)
- **Git** (optional but recommended)

Verify:

```bash
docker --version
docker compose version
```

---

## Quick start (recommended: official Docker Compose stack)

### 1) Get the Compose project

```bash
git clone https://github.com/opf/openproject-docker-compose.git --depth=1 --branch=stable/16 openproject
```

> You will get the stable/16 bracnh.

---

### 2) Create your environment file
```
cd openproject
```

```bash
cp .env.example .env
```

Open `.env` and set these values (important for localhost):

```env
# Base URL used by OpenProject to generate links
OPENPROJECT_HOST__NAME=localhost:8080

# For localhost HTTP (no TLS)
OPENPROJECT_HTTPS=false

# Bind to localhost only (NOT reachable from LAN)
PORT=127.0.0.1:8080
```

> If `PORT` is left as `127.0.0.1:8080`, other machines **cannot** access it (this is what we want for a local-only setup).

**(sample .env file)**
```env
TAG=16-slim
OPENPROJECT_HTTPS=false
OPENPROJECT_HOST__NAME=localhost:8080
PORT=127.0.0.1:8080
OPENPROJECT_RAILS__RELATIVE__URL__ROOT=
IMAP_ENABLED=false
POSTGRES_PASSWORD=SomeStrongPasswordHere
DATABASE_URL=postgres://postgres:SomeStrongPasswordHere@db/openproject?pool=20&encoding=unicode&reconnect=true
RAILS_MIN_THREADS=4
RAILS_MAX_THREADS=16
PGDATA=pgdata
OPDATA=opdata
```

---

### 3) Create persistent data folders (on the host)

From the project directory:

**Linux/macOS (bash):**
```bash
mkdir -p ./pgdata ./assets
```

**Windows PowerShell:**
```powershell
New-Item -ItemType Directory -Force -Path .\pgdata,.\assets | Out-Null
```

---

### 3.5) Create the hostname mapping (Windows)

**(local-only)**: map to 127.0.0.1:

- Open Notepad as Administrator and Open file:
```
C:\Windows\System32\drivers\etc\hosts
```
- Add this line:
```
127.0.0.1  localhost
```
- Save

- Flush DNS:
```powershell
ipconfig /flushdns
```
- Test:
```powershell
ping localhost
```

---

### 4) Start OpenProject

```bash
docker compose up -d

or

docker compose up -d --build --pull always
```

Check containers:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f
```

---

### 5) Open in your browser

Go to:

- **http://localhost:8080**

On first launch, OpenProject may take a minute while it initializes the database and assets.

- login with 'admin' and password.

---

## Useful commands

### Stop / start

```bash
docker compose stop
docker compose start
```

### Shut down (keeps data)

```bash
docker compose down
```

### Full reset (⚠️ deletes all OpenProject data)

```bash
docker compose down -v
```

Also remove the local folders if you created them:

```bash
rm -rf pgdata assets
```

(Windows PowerShell)
```powershell
Remove-Item -Recurse -Force .\pgdata,.\assets
```

### Run a shell inside the web container

```bash
docker compose exec web bash
```

### Rails database migration check (optional)

```bash
docker compose exec web bash -lc "bin/rails db:abort_if_pending_migrations RAILS_ENV=production"
```

---

## Troubleshooting

### Port 8080 is already in use
- Change `PORT=127.0.0.1:8080` to another port, e.g. `127.0.0.1:8085`
- Then restart:

```bash
docker compose down
docker compose up -d
```

### Containers start but the UI doesn’t load
- Watch logs:

```bash
docker compose logs -f
```

- Give it a bit on first run (DB init + migrations).

### Email/SMTP
For localhost testing, you can skip SMTP. For professional use (LAN/production), configure SMTP later with your organization’s mail relay/service account.

---

## Next step
If you want teammates to access OpenProject from their PCs, you’ll move from **localhost** to a **LAN deployment** with:
- static IP / DNS name
- firewall rules
- (recommended) HTTPS reverse proxy
- SMTP relay for invites/notifications

