# Project 07 — WordPress + MySQL Stack with Volume Backup & Restore Drill

| | |
|---|---|
| **Module** | 4 — Docker II |
| **Level** | L2 — Engineer |
| **Role you play** | DevOps Engineer responsible for the marketing website platform |
| **Scope** | `1-Docker-storage.md`, `2-Linking-containers.md`, `3/4-Docker-compose`, `5-Deploying-wordpress.md` |

---

## 1. Business problem

Marketing runs the company website on WordPress. It currently lives on one hand-built VM. Two
real incidents have already happened: a developer `docker rm`'d the database container and lost
three days of content, and nobody can bring up an identical copy for staging.

You must deliver the stack as **code** (Docker Compose), with **persistent, backed-up storage**,
and — most importantly — **prove the restore works** in a timed drill. A backup that has never
been restored is not a backup.

## 2. Objectives

1. Run WordPress + MySQL with Compose v2 on a user-defined network.
2. Keep all state in named volumes; prove containers are disposable.
3. Implement a scripted backup of the database **and** the uploads volume.
4. Run a **destructive restore drill** and measure RTO (recovery time) and RPO (data loss).
5. Document the operational runbook for the on-call engineer.

## 3. Scope

**In scope:** Docker volumes (named, bind, `--volumes-from`), user-defined bridge networks and
DNS-based service discovery, `docker compose` v2 (`up/down/ps/logs/exec/restart`), environment
files, healthchecks, `depends_on`.

**Out of scope:** Swarm/HA (Project 08), Kubernetes, TLS certificates, CDN.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** APP-SOP-007 · **Title:** Operate and recover the WordPress platform

### 4.1 Storage design

| Data | Where | Volume | Backup frequency | RPO |
|---|---|---|---|---|
| WordPress core + plugins + themes | container `/var/www/html` | `wp_data` | daily | 24 h |
| Uploaded media | inside `wp_data/wp-content/uploads` | `wp_data` | daily | 24 h |
| Database | container `/var/lib/mysql` | `db_data` | **hourly** logical dump | 1 h |
| Compose/config | Git | — | on commit | 0 |

**Rule:** no business data in the container writable layer. If `docker rm` loses it, the design
is wrong.

### 4.2 Operational rules

1. `docker compose down` is safe. **`docker compose down -v` destroys volumes** — it requires a
   ticket and a verified backup.
2. Credentials live in `.env`, which is `.gitignore`d; only `.env.example` is committed.
3. The database is never published to the host — only the web tier is.
4. Backups are verified by restore, monthly, into a throwaway stack.
5. Every change to `docker-compose.yml` goes through Git.

### 4.3 Backup procedure (hourly, cron)

| # | Step |
|---|---|
| 1 | `mysqldump --single-transaction` from the db container to a timestamped `.sql.gz` |
| 2 | `tar` the `wp_data` volume via a throwaway helper container |
| 3 | Write both to `/opt/backups/wordpress/<timestamp>/` |
| 4 | Record a SHA256 checksum |
| 5 | Prune backups older than 7 days |
| 6 | Log success/failure; a silent failure is the real risk |

### 4.4 Restore procedure (RTO target: 15 minutes)

| # | Step |
|---|---|
| 1 | Declare the incident, note the time |
| 2 | Stop the stack (`docker compose down`, **without** `-v`) |
| 3 | Pick the newest backup with a valid checksum |
| 4 | Recreate volumes and restore the media tar |
| 5 | Start only the DB, wait for healthy, import the SQL dump |
| 6 | Start the web tier |
| 7 | Validate: site loads, a known post exists, media renders, admin login works |
| 8 | Record RTO and actual data loss; raise follow-up actions |

---

## 5. Hands-on implementation

### Step 1 — Prerequisites

```bash
docker --version && docker compose version     # Compose v2 plugin (see Module 3 install)
mkdir -p ~/wordpress-platform && cd ~/wordpress-platform
```

### Step 2 — Understand volumes before using them (storage drill)

```bash
# Anonymous vs named vs bind
docker volume create demo_named
docker run -d --name v1 -v demo_named:/data alpine sleep 600
docker exec v1 sh -c 'echo "survives" > /data/file.txt'

docker rm -f v1
docker run --rm -v demo_named:/data alpine cat /data/file.txt   # still there

docker volume ls
docker volume inspect demo_named          # see the host Mountpoint
docker volume rm demo_named
```

```bash
# Bind mount: host path, useful for config and dev
mkdir -p /tmp/bindtest && echo "from host" > /tmp/bindtest/a.txt
docker run --rm -v /tmp/bindtest:/mnt:ro alpine cat /mnt/a.txt
docker run --rm -v /tmp/bindtest:/mnt:ro alpine \
  sh -c 'touch /mnt/b.txt 2>&1 || echo "read-only mount enforced"'
```

### Step 3 — Understand container linking / DNS

```bash
docker network create wp-lab
docker run -d --name db-lab --network wp-lab -e MYSQL_ROOT_PASSWORD=x mysql:8.4
docker run --rm --network wp-lab alpine sh -c 'nslookup db-lab || getent hosts db-lab'
docker rm -f db-lab && docker network rm wp-lab
```

> **Job insight:** on a user-defined bridge network, containers resolve each other by
> **service/container name**. The legacy `--link` flag is deprecated — never use it in new work.

### Step 4 — The stack as code

`.env` (never committed):

```bash
cat > .env <<'EOF'
MYSQL_ROOT_PASSWORD=<STRONG_ROOT_PASSWORD>
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
MYSQL_PASSWORD=<STRONG_APP_PASSWORD>
WP_PORT=8080
EOF
chmod 600 .env

sed 's/=.*/=CHANGE_ME/' .env > .env.example

cat > .gitignore <<'EOF'
.env
backups/
EOF
```

`docker-compose.yml` — note: **no `version:` key**, it is obsolete in Compose v2:

```yaml
services:
  db:
    image: mysql:8.4
    container_name: wp-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wpnet
    # deliberately NOT published to the host
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 40s

  wordpress:
    image: wordpress:php8.3-apache
    container_name: wp-web
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - wp_data:/var/www/html
    ports:
      - "${WP_PORT}:80"
    networks:
      - wpnet
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost/ >/dev/null || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

volumes:
  db_data:
  wp_data:

networks:
  wpnet:
    driver: bridge
```

### Step 5 — Bring it up and validate

```bash
docker compose config            # validate and show resolved values
docker compose up -d
docker compose ps
docker compose logs -f --tail 30 wordpress    # Ctrl-C when ready

curl -sI http://localhost:8080 | head -1
docker inspect --format '{{.State.Health.Status}}' wp-db wp-web
docker volume ls | grep wordpress-platform
docker network inspect wordpress-platform_wpnet --format '{{json .Containers}}' | jq 'keys'
```

Open `http://<VM-external-IP>:8080`, complete the WordPress installer, publish a post titled
**"Backup Drill Marker"**, and upload an image. That content is your restore proof.

### Step 6 — Prove the containers are disposable

```bash
docker compose down              # containers removed, volumes kept
docker volume ls                 # volumes still present
docker compose up -d
curl -s http://localhost:8080 | grep -o "Backup Drill Marker"   # content survived
```

### Step 7 — Backup script

```bash
mkdir -p ~/wordpress-platform/scripts
cat > ~/wordpress-platform/scripts/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
set -a; . ./.env; set +a

STAMP=$(date +%Y%m%d-%H%M%S)
DEST="/opt/backups/wordpress/${STAMP}"
sudo mkdir -p "$DEST" && sudo chown "$USER" "$DEST"

echo "[1/4] database dump"
docker compose exec -T db \
  mysqldump --single-transaction --routines --triggers \
  -u root -p"${MYSQL_ROOT_PASSWORD}" "${MYSQL_DATABASE}" \
  | gzip > "${DEST}/db.sql.gz"

echo "[2/4] wp_data volume"
docker run --rm \
  -v wordpress-platform_wp_data:/src:ro \
  -v "${DEST}":/backup \
  alpine tar czf /backup/wp_data.tar.gz -C /src .

echo "[3/4] checksums"
( cd "$DEST" && sha256sum db.sql.gz wp_data.tar.gz > SHA256SUMS )

echo "[4/4] prune older than 7 days"
sudo find /opt/backups/wordpress -maxdepth 1 -type d -mtime +7 -exec rm -rf {} +

ls -lh "$DEST"
echo "BACKUP OK: $DEST"
EOF
chmod +x ~/wordpress-platform/scripts/backup.sh
~/wordpress-platform/scripts/backup.sh
```

Schedule it hourly:

```bash
( crontab -l 2>/dev/null; echo "0 * * * * $HOME/wordpress-platform/scripts/backup.sh >> /var/tmp/wp-backup.log 2>&1" ) | crontab -
crontab -l
```

### Step 8 — THE DRILL: destroy everything, then restore

```bash
# Note the start time - this is your RTO clock
date

# Simulated disaster: volumes destroyed
docker compose down -v
docker volume ls | grep wordpress-platform || echo "volumes gone"
```

**Restore:**

```bash
cd ~/wordpress-platform
BK=$(ls -1d /opt/backups/wordpress/*/ | tail -1); echo "$BK"
( cd "$BK" && sha256sum -c SHA256SUMS )          # verify integrity FIRST

# Recreate volumes
docker volume create wordpress-platform_db_data
docker volume create wordpress-platform_wp_data

# Restore site files
docker run --rm \
  -v wordpress-platform_wp_data:/dest \
  -v "${BK}":/backup:ro \
  alpine sh -c 'cd /dest && tar xzf /backup/wp_data.tar.gz'

# Start DB only and wait for it to be healthy
docker compose up -d db
until [ "$(docker inspect --format '{{.State.Health.Status}}' wp-db)" = "healthy" ]; do
  sleep 3; echo "waiting for db..."
done

# Import the dump
set -a; . ./.env; set +a
gunzip -c "${BK}/db.sql.gz" | \
  docker compose exec -T db mysql -u root -p"${MYSQL_ROOT_PASSWORD}" "${MYSQL_DATABASE}"

# Bring the web tier up
docker compose up -d
docker compose ps
```

**Validate and close the drill:**

```bash
date                                                   # stop the RTO clock
curl -s http://localhost:8080 | grep -o "Backup Drill Marker"
docker compose exec -T db mysql -u root -p"${MYSQL_ROOT_PASSWORD}" \
  -e "SELECT post_title,post_date FROM ${MYSQL_DATABASE}.wp_posts WHERE post_status='publish';"
```

Record in the drill report: start time, end time, **RTO achieved**, **actual data loss (RPO)**,
and anything that slowed you down.

---

## 6. Acceptance criteria

- [ ] Stack starts from `docker compose up -d` with no manual steps
- [ ] Database port is **not** published to the host
- [ ] Credentials only in `.env`; `.env.example` committed, `.env` ignored
- [ ] Both services report `healthy`
- [ ] `docker compose down` + `up` preserves content
- [ ] Backup produces db dump + media tar + checksums
- [ ] Full restore completed after `down -v` with the marker post and media intact
- [ ] RTO and RPO measured and written down

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| WordPress: "Error establishing a database connection" | DB not ready yet | `depends_on: condition: service_healthy` + healthcheck (already in the file) |
| `mysqldump: Got error: 1045 Access denied` | Wrong password / `.env` not loaded | `set -a; . ./.env; set +a`, verify with `docker compose config` |
| Volume name not found on restore | Compose prefixes volumes with the project name | `docker volume ls`; project = directory name, or set `name:` in the file |
| Restored site shows "install" screen | DB imported into the wrong database or dump empty | Check dump size, confirm `MYSQL_DATABASE` matches |
| Media missing after restore | Only the DB was backed up | Media lives in `wp_data` — back up both |
| Permission denied writing uploads | Volume owned by the wrong UID | `docker compose exec wordpress chown -R www-data:www-data /var/www/html/wp-content` |
| `docker compose` not found | Using the obsolete v1 binary | Install `docker-compose-plugin`, use `docker compose` (with a space) |
| Port 8080 already allocated | Another process | `ss -tulpn | grep 8080`, change `WP_PORT` |

---

## 8. Cleanup

```bash
cd ~/wordpress-platform
docker compose down -v
docker volume prune -f
sudo rm -rf /opt/backups/wordpress
crontab -l | grep -v wordpress-platform | crontab -
```

---

## 9. Deliverables

1. Repo: `docker-compose.yml`, `.env.example`, `.gitignore`, `scripts/backup.sh`.
2. `RUNBOOK-APP-007.md` — storage design table, operational rules, backup + restore procedures.
3. `DR-DRILL-REPORT.md` — timestamps, measured RTO/RPO, screenshots before and after restore.
4. Evidence: `docker compose ps`, healthy statuses, checksum verification output.

---

## 10. Interview talking points

* *"How do you persist data in Docker?"* — Named volumes for application state, bind mounts for
  config; nothing business-critical in the container writable layer.
* *"How do containers find each other?"* — User-defined bridge network, DNS by service name;
  `--link` is deprecated.
* *"How do you know your backups work?"* — Because I run restore drills and measure RTO/RPO;
  an untested backup is an assumption.
* *"Why isn't the database port published?"* — It only needs to be reachable on the internal
  Compose network; publishing it exposes it to the internet.
* *"What is the danger of `docker compose down -v`?"* — It deletes volumes. It should require a
  ticket and a verified backup.

**CV bullet:** *Migrated the corporate WordPress platform to a code-defined Docker Compose stack
with named-volume persistence, hourly verified backups and a documented restore runbook;
validated recovery through destructive DR drills with measured RTO/RPO.*
