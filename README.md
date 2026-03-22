# Docker compose for PostgreSQL and n8n

This guide uses Docker Compose with **host-mounted folders** so your PostgreSQL, pgAdmin 4, and n8n data all live in regular directories on your host machine. That makes it easier to inspect files and create backups.

---

## 📁 Directory Structure

```bash
~/docker/postgres/            ← Shared PostgreSQL stack
├── docker-compose.yml
├── .env
└── data/                     ← PostgreSQL data stored here

~/docker/pgadmin/             ← pgAdmin 4 stack
├── docker-compose.yml
└── data/                     ← pgAdmin data stored here

~/docker/n8n/                 ← n8n stack
├── docker-compose.yml
├── .env
└── data/                     ← n8n data stored here
```

---

## 🐘 Step 1: Set Up PostgreSQL

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
      - ./data:/var/lib/postgresql/data
    networks:
      - shared_db
    ports:
      - "5432:5432"
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

### 1.4 Start PostgreSQL

```bash
docker compose up -d
```

After startup, PostgreSQL data files will appear in `~/docker/postgres/data/`.

### 1.5 Create a dedicated database and user for n8n

```bash
docker compose exec postgres psql -U admin -c "CREATE DATABASE n8n;"
docker compose exec postgres psql -U admin -c "CREATE USER n8n WITH PASSWORD 'n8n_password';"
docker compose exec postgres psql -U admin -c "GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;"
```

> ⚠️ **PostgreSQL 15+ requires explicit schema permissions.** Run these additional commands or n8n may fail with `permission denied for schema public`:

```bash
docker compose exec postgres psql -U admin -d n8n -c "GRANT ALL ON SCHEMA public TO n8n;"
docker compose exec postgres psql -U admin -d n8n -c "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO n8n;"
docker compose exec postgres psql -U admin -d n8n -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO n8n;"
```

> 💡 Repeat these database/user creation steps for every new project that needs its own database.

---

## 🐘 Step 2: Set Up pgAdmin 4

If you want a browser-based PostgreSQL GUI similar to SQL Developer, you can run **pgAdmin 4** in its own Docker Compose stack.

### 2.1 Create the directories

```bash
mkdir -p ~/docker/pgadmin/data && cd ~/docker/pgadmin
```

### 2.2 Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  pgadmin:
    image: dpage/pgadmin4:latest
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: your_pgadmin_password
    ports:
      - "8080:80"
    volumes:
      - ./data:/var/lib/pgadmin
    networks:
      - shared_db

networks:
  shared_db:
    external: true
```

### 2.3 Fix folder permissions

pgAdmin runs as user `pgadmin` (UID 5050) inside the container. The mounted folder must be writable by that user:

```bash
sudo chown -R 5050:5050 ~/docker/pgadmin/data
```

### 2.4 Start pgAdmin 4

```bash
docker compose up -d
```

pgAdmin 4 is now available at: [http://localhost:8080](http://localhost:8080)

### 2.5 Log in to pgAdmin

Use the credentials from your `docker-compose.yml`:

| Field | Value |
|---|---|
| Email | `admin@example.com` |
| Password | `your_pgadmin_password` |

### 2.6 Add your PostgreSQL server in pgAdmin

After logging in:

1. Right-click **Servers**
2. Select **Register > Server...**
3. In the **General** tab:
   - Name: `Local Postgres`
4. In the **Connection** tab:
   - Host name/address: `postgres`
   - Port: `5432`
   - Maintenance database: `postgres`
   - Username: `admin`
   - Password: value of `POSTGRES_PASSWORD`

> ✅ Because pgAdmin is connected to the same `shared_db` Docker network, use `postgres` as the hostname, **not** `localhost`.

---

## ⚙️ Step 3: Set Up n8n

### 3.1 Create the directories

```bash
mkdir -p ~/docker/n8n/data && cd ~/docker/n8n
```

### 3.2 Fix folder permissions

n8n runs as user `node` (UID 1000) inside the container. The mounted folder must be writable by that user:

```bash
chown -R 1000:1000 ~/docker/n8n/data
```

### 3.3 Create the `.env` file

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

> 🔴 **Back up `N8N_ENCRYPTION_KEY` somewhere safe!** Losing it means losing access to all stored credentials.

### 3.4 Create `docker-compose.yml`

```yaml
version: '3.8'

networks:
  shared_db:
    external: true

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
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
    volumes:
      - ./data:/home/node/.n8n
    networks:
      - shared_db
```

### 3.5 Start n8n

```bash
docker compose up -d
```

n8n is now available at: [http://localhost:5678](http://localhost:5678)

---

## 🖥️ Step 4: Connect via a GUI Tool (Optional)

Since port `5432` is exposed to the host, you can also connect using any PostgreSQL GUI tool installed locally:

| Tool | Free | Notes |
|---|---|---|
| **DBeaver** | ✅ | Best all-around, supports many DBs |
| **pgAdmin 4** | ✅ | Official PostgreSQL GUI |
| **TablePlus** | Freemium | Clean UI, Mac/Windows/Linux |
| **Beekeeper Studio** | ✅ | Lightweight, open source |
| **DataGrip** | ❌ | JetBrains, paid but excellent |

**Connection settings for local desktop tools:**

| Field | Value |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | `n8n` |
| Username | `n8n` (or `admin` for full access) |
| Password | as set in `.env` |

---

## 🔄 Step 5: Adding More Projects

For each new project that needs PostgreSQL:

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
         - ./data:/app/data
       networks:
         - shared_db
   ```

---

## 💾 Data Storage

All data lives directly on your host — no Docker volume commands needed:

| Data | Host Path |
|---|---|
| PostgreSQL databases | `~/docker/postgres/data/` |
| pgAdmin settings | `~/docker/pgadmin/data/` |
| n8n workflows & settings | `~/docker/n8n/data/` |
| n8n encryption key | `~/docker/n8n/.env` → `N8N_ENCRYPTION_KEY` |

---

## 🗄️ Backup — As Simple As Copying a Folder

Because everything is on the host filesystem, backup is straightforward:

### Back up PostgreSQL data

```bash
# Option A: copy the raw data folder (stop PostgreSQL first for consistency)
cd ~/docker/postgres
docker compose stop
cp -r ~/docker/postgres/data ~/backups/postgres_$(date +%Y%m%d)
docker compose start

# Option B: pg_dump (no downtime needed)
cd ~/docker/postgres
docker compose exec postgres \
  pg_dump -U n8n n8n > ~/backups/n8n_db_$(date +%Y%m%d).sql
```

### Back up pgAdmin data

```bash
cp -r ~/docker/pgadmin/data ~/backups/pgadmin_$(date +%Y%m%d)
```

### Back up n8n data

```bash
cp -r ~/docker/n8n/data ~/backups/n8n_$(date +%Y%m%d)
cp ~/docker/n8n/.env ~/backups/n8n_env_$(date +%Y%m%d)
```

### What to back up

| Item | Host Path | Priority |
|---|---|---|
| PostgreSQL data folder | `~/docker/postgres/data/` | 🔴 Critical |
| pgAdmin data folder | `~/docker/pgadmin/data/` | 🟠 High |
| `N8N_ENCRYPTION_KEY` | `~/docker/n8n/.env` | 🔴 Critical |
| n8n data folder | `~/docker/n8n/data/` | 🔴 Critical |
| `.env` files | `~/docker/postgres/.env`, `~/docker/n8n/.env` | 🟠 High |
| `docker-compose.yml` files | `~/docker/postgres/`, `~/docker/pgadmin/`, `~/docker/n8n/` | 🟡 Medium |

> ✅ No special Docker commands needed — just regular file copies.

---

## 🔁 Updating the containers

### Update PostgreSQL

```bash
cd ~/docker/postgres
docker compose pull
docker compose up -d
```

### Update pgAdmin 4

```bash
cd ~/docker/pgadmin
docker compose pull
docker compose up -d
```

### Update n8n

```bash
cd ~/docker/n8n
docker compose pull
docker compose up -d
```

Your data in the mounted host folders is preserved during updates.

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

```text
Host Machine
│
├── ~/docker/postgres/
│   ├── data/                  ← PostgreSQL files (directly on host)
│   └── postgres (container) ───────────────┐
│                                           │
├── ~/docker/pgadmin/
│   ├── data/                  ← pgAdmin files (directly on host)
│   └── pgadmin (container) ────────────────┤
│                                           │ shared_db network
├── ~/docker/n8n/
│   ├── data/                  ← n8n files (directly on host)
│   └── n8n (container) ────────────────────┘
│
├── Port 5432 exposed → PostgreSQL access for local GUI tools
├── Port 8080 exposed → pgAdmin 4 UI (http://localhost:8080)
└── Port 5678 exposed → n8n UI (http://localhost:5678)
```
