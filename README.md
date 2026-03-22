# n8n with Shared PostgreSQL via Docker Compose (Host-Mounted Folders)

This guide is a variant of the standard Docker volume setup. Instead of using named Docker volumes, **all data is stored in regular folders on your host machine** — making backups as simple as copying a folder.

---

## 📁 Directory Structure

```
~/docker/postgres/            ← Shared Postgres stack
├── docker-compose.yml
├── .env
└── data/                     ← PostgreSQL data stored here

~/docker/n8n/                 ← n8n stack
├── docker-compose.yml
├── .env
└── data/                     ← n8n data stored here
```

---

## 🐘 Step 1: Set Up the Shared PostgreSQL Stack

### 1.1 Create the directories

```bash
mkdir -p ~/docker/postgres/data && cd ~/docker/postgres
```

### 1.2 Create the `.env` file

```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=your_strong_password
```

> ⚠️ **Never commit this file to version control.** Add `.env` to your `.gitignore`.

### 1.3 Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: postgres
    volumes:
      - ./data:/var/lib/postgresql/data   # ← host folder mount
    networks:
      - shared_db
    ports:
      - "5432:5432"   # Expose to host for GUI tools (DBeaver, pgAdmin, etc.)
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER}']
      interval: 5s
      timeout: 5s
      retries: 10

networks:
  shared_db:
    name: shared_db
    driver: bridge
```

### 1.4 Start the shared Postgres

```bash
docker compose up -d
```

After startup, you will see the Postgres data files appear in `~/docker/postgres/data/`.

### 1.5 Create a dedicated database and user for n8n

```bash
docker compose exec postgres psql -U admin -c "CREATE DATABASE n8n;"
docker compose exec postgres psql -U admin -c "CREATE USER n8n WITH PASSWORD 'n8n_password';"
docker compose exec postgres psql -U admin -c "GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;"
```

> ⚠️ **Postgres 15+ requires explicit schema permissions.** Run these additional commands or n8n will fail with `permission denied for schema public`:

```bash
docker compose exec postgres psql -U admin -d n8n -c "GRANT ALL ON SCHEMA public TO n8n;"
docker compose exec postgres psql -U admin -d n8n -c "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO n8n;"
docker compose exec postgres psql -U admin -d n8n -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO n8n;"
```

> 💡 Repeat **all** steps in 1.5 for every new project that needs its own database.

---

## ⚙️ Step 2: Set Up n8n

### 2.1 Create the directories

```bash
mkdir -p ~/docker/n8n/data && cd ~/docker/n8n
```

### 2.2 Fix folder permissions

n8n runs as user `node` (UID 1000) inside the container. The mounted folder must be writable by that user:

```bash
chown -R 1000:1000 ~/docker/n8n/data
```

### 2.3 Create the `.env` file

```env
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
N8N_ENCRYPTION_KEY=your_random_32char_key_here
GENERIC_TIMEZONE=Europe/Zurich

DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=n8n_password
```

> 🔴 **Back up `N8N_ENCRYPTION_KEY` somewhere safe!** Losing it means losing access to all stored credentials. Since the data folder is on your host, you can simply copy `~/docker/n8n/.env` to a safe location.

### 2.4 Create `docker-compose.yml`

```yaml
version: '3.8'

networks:
  shared_db:
    external: true   # Joins the shared Postgres network

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=${N8N_PORT}
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres        # Service name of the shared Postgres container
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
    volumes:
      - ./data:/home/node/.n8n        # ← host folder mount
    networks:
      - shared_db
```

### 2.5 Start n8n

```bash
docker compose up -d
```

n8n is now available at: [http://localhost:5678](http://localhost:5678)

---

## 🖥️ Step 3: Connect via a GUI Tool (Optional)

Since port `5432` is exposed to the host, you can connect using any PostgreSQL GUI tool:

| Tool | Free | Notes |
|---|---|---|
| **DBeaver** | ✅ | Best all-around, supports many DBs |
| **pgAdmin 4** | ✅ | Official PostgreSQL GUI |
| **TablePlus** | Freemium | Clean UI, Mac/Windows/Linux |
| **Beekeeper Studio** | ✅ | Lightweight, open source |
| **DataGrip** | ❌ | JetBrains, paid but excellent |

**Connection settings:**

| Field | Value |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | `n8n` |
| Username | `n8n` (or `admin` for full access) |
| Password | as set in `.env` |

---

## 🔄 Step 4: Adding More Projects

For each new project that needs Postgres:

1. Create a new DB and user:
   ```bash
   cd ~/docker/postgres
   docker compose exec postgres psql -U admin -c "CREATE DATABASE myapp;"
   docker compose exec postgres psql -U admin -c "CREATE USER myapp WITH PASSWORD 'myapp_password';"
   docker compose exec postgres psql -U admin -c "GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp;"
   ```

2. Create a data folder for the new project:
   ```bash
   mkdir -p ~/myapp/data
   ```

3. In the new project's `docker-compose.yml`, join the shared network and mount a host folder:
   ```yaml
   networks:
     shared_db:
       external: true

   services:
     myapp:
       ...
       volumes:
         - ./data:/app/data    # adjust path to match the app
       networks:
         - shared_db
   ```

---

## 💾 Data Storage

All data lives directly on your host — no Docker volume commands needed:

| Data | Host Path |
|---|---|
| n8n workflows & settings | `~/docker/n8n/data/` |
| PostgreSQL databases | `~/docker/postgres/data/` |
| Encryption key | `~/docker/n8n/.env` → `N8N_ENCRYPTION_KEY` |

---

## 🗄️ Backup — As Simple As Copying a Folder

Because everything is on the host filesystem, backup is straightforward:

### Back up n8n data
```bash
cp -r ~/n8n/data/n8n ~/backups/n8n_$(date +%Y%m%d)
cp ~/n8n/.env ~/backups/n8n_env_$(date +%Y%m%d)
```

### Back up PostgreSQL data
```bash
# Option A: copy the raw data folder (stop Postgres first for consistency)
docker compose -f ~/docker-shared/docker-compose.yml stop
cp -r ~/docker-shared/data/postgres ~/backups/postgres_$(date +%Y%m%d)
docker compose -f ~/docker-shared/docker-compose.yml start

# Option B: pg_dump (no downtime needed)
docker compose -f ~/docker-shared/docker-compose.yml exec postgres \
  pg_dump -U n8n n8n > ~/backups/n8n_db_$(date +%Y%m%d).sql
```

### What to back up

| Item | Host Path | Priority |
|---|---|---|
| `N8N_ENCRYPTION_KEY` | `~/n8n/.env` | 🔴 Critical |
| n8n data folder | `~/n8n/data/n8n/` | 🔴 Critical |
| Postgres data folder | `~/docker-shared/data/postgres/` | 🔴 Critical |
| `.env` files | `~/n8n/.env`, `~/docker-shared/.env` | 🟠 High |
| `docker-compose.yml` files | `~/n8n/`, `~/docker-shared/` | 🟡 Medium |

> ✅ No special Docker commands needed — just regular file copies!

---

## 🔁 Updating n8n

```bash
cd ~/n8n
docker compose pull
docker compose up -d
```

Your data in `~/n8n/data/n8n/` is untouched during updates.

---

## ⚖️ Host Mounts vs. Named Volumes

| | Host Mounts (this guide) | Named Docker Volumes |
|---|---|---|
| Data location | Visible folder on host | Hidden in Docker internals |
| Backup | Copy folder | Requires `docker cp` or CLI export |
| Accidental deletion risk | Low (standard file ops) | High (`docker compose down -v`) |
| Portability | Tied to host path | More portable across machines |
| Permissions | May need `chown` | Handled automatically |
| Best for | Local dev / home lab ✅ | Production / multi-host setups |

---

## 🏗️ Architecture Overview

```
Host Machine
│
├── ~/docker-shared/
│   ├── data/postgres/        ← PostgreSQL files (directly on host)
│   └── postgres (container)  ←─── shared_db network ───────┐
│                                                           │
├── ~/n8n/                                                  │
│   ├── data/n8n/             ← n8n files (directly on host)│
│   └── n8n (container) ────────────────────────────────────┘
│
└── Port 5432 exposed → GUI tools (DBeaver, pgAdmin, etc.)
    Port 5678 exposed → n8n UI (http://localhost:5678)
```