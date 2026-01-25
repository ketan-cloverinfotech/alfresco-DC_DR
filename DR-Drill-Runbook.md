# Alfresco Community (Docker Compose) — DR Drill Runbook (DC ↔ DR)

**Environment**
- DC (Primary normally): `10.128.0.17`
- DR (Standby normally): `10.128.0.18`
- Stack: PostgreSQL 16.5 + Alfresco Repo + Share + Solr + ActiveMQ + Transform + Nginx proxy (+ OCR)
- Deployment: Docker Compose on both servers
- Data paths (host):
  - Postgres data: `/root/volumes/data/postgres-data`
  - Alfresco alf_data: `/root/volumes/data/alf-repo-data`
    - Content: `/root/volumes/data/alf-repo-data/contentstore`
    - Deleted: `/root/volumes/data/alf-repo-data/contentstore.deleted`
  - Config:
    - `/root/volumes/config/postgres/postgresql.conf`
    - `/root/volumes/config/postgres/pg_hba.conf`
    - `/root/volumes/config/postgres/init/01-replication.sql`

---

## 0) Goals and Rules

### Goal
Perform a **planned failover drill**:
1) Promote DR to PRIMARY (DC ➜ DR)  
2) Rebuild DC as STANDBY of DR (reverse replication)  
3) Ensure contentstore sync continues in the correct direction (PRIMARY ➜ STANDBY)

### Golden Rules (VERY IMPORTANT)
1. **Only ONE PostgreSQL primary at any time.**
2. **Never run “write workloads” on the standby.**
3. **Never rsync FROM standby TO primary** (this caused content loss before).
   - Contentstore sync must be **PRIMARY ➜ STANDBY** only.

---

## 1) Pre-requisites Checklist

### 1.1 Network
- DC can reach DR on `5432` (Postgres) and SSH for rsync.
- DR can reach DC if needed (optional).
- Time sync (NTP) on both hosts.

### 1.2 PostgreSQL replication config (on whoever is PRIMARY)
Ensure in `postgresql.conf` (mapped into container) at least:
- `listen_addresses='*'`
- `wal_level=replica`
- `max_wal_senders >= 5`
- `max_replication_slots >= 5`
- `hot_standby=on` (safe even on primary)
- Optional: `wal_keep_size` (tune)

### 1.3 pg_hba.conf rules
On PRIMARY, allow:
- Replication user from standby IP (example pattern)
- Alfresco DB user from docker compose network (e.g. `172.18.0.0/16` or `172.19.0.0/16`) and/or host subnet (e.g. `10.128.0.0/16`)

### 1.4 Replication user + slot
On PRIMARY ensure user exists and create slot:
- user: `replicator`
- password: `StrongPass@123`
- slot name:
  - When DC is primary and DR is standby: `dr_slot`
  - When DR is primary and DC is standby: `dc_slot`

### 1.5 Contentstore replication method (rsync)
- SSH key auth user (example: `drsync`)
- Decide direction:
  - When DC is primary → sync **DC ➜ DR**
  - When DR is primary → sync **DR ➜ DC**
- Run rsync cron **ONLY on the standby (pull model is safest)**:
  - Standby pulls from Primary, never writes into Primary.

---

## 2) Standard Verification Commands (Use anytime)

### 2.1 Identify Primary vs Standby
Run on a node:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_is_in_recovery();"
```
- `f` = PRIMARY
- `t` = STANDBY

### 2.2 On PRIMARY: replication status / lag
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT client_addr, application_name, state, sync_state,
       sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       reply_time
FROM pg_stat_replication;"
```
- Expect a row for standby IP
- `state=streaming`
- `lag_bytes` ideally `0`

### 2.3 On STANDBY: WAL receiver status (PostgreSQL 16 compatible)
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT status, conninfo, receive_start_lsn, written_lsn, flushed_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_wal_receiver;"
```
- `status=streaming`
- `conninfo` contains PRIMARY IP

---

## 3) DR Drill — Planned Failover (DC Primary ➜ DR Primary)

> Use this when DC is primary and you want to switch production to DR.

### Step 3.1 Confirm current roles
- On DC: should be `f`
- On DR: should be `t`

### Step 3.2 Freeze writes on DC (Stop Alfresco stack)
On **DC** stop all non-postgres services:
```bash
cd /root
docker compose stop proxy content-app share alfresco transform-core-aio ocr-transformer solr6 activemq
```
(Leave postgres running for replication catch-up)

### Step 3.3 Wait until DR catches up (Lag = 0)
On **DC (PRIMARY)**:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT client_addr, state,
       sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;"
```

On **DR (STANDBY)**:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT status, latest_end_lsn, latest_end_time
FROM pg_stat_wal_receiver;"
```

### Step 3.4 Final content sync (DC ➜ DR) — One last time
Run a final rsync for:
- `contentstore`
- `contentstore.deleted`
- (optional) `contentstore.spacesStore` if used

Example (adjust ssh user/key/path):
```bash
rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2   -e "ssh -i /root/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new"   /root/volumes/data/alf-repo-data/contentstore/   drsync@10.128.0.18:/root/volumes/data/alf-repo-data/contentstore/

rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2   -e "ssh -i /root/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new"   /root/volumes/data/alf-repo-data/contentstore.deleted/   drsync@10.128.0.18:/root/volumes/data/alf-repo-data/contentstore.deleted/
```

### Step 3.5 Fence DC (STOP DC Postgres) — Prevent split-brain
On **DC**:
```bash
cd /root
docker compose stop postgres
```

### Step 3.6 Promote DR to PRIMARY
On **DR**:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_promote(wait_seconds => 60);"
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_is_in_recovery();"
```
Expected: `f`

### Step 3.7 Start full Alfresco stack on DR (New Production)
On **DR**:
```bash
cd /root
docker compose up -d
docker compose ps
```

---

## 4) Post-Failover — Rebuild DC as Standby of DR (Reverse Replication)

> Use this after DR is primary and you want DC to be its standby.

### Step 4.1 On DR (PRIMARY): allow DC replication + create slot
1) Update DR `pg_hba.conf` to allow DC:
```conf
# DC (10.128.0.17) connects to DR primary
host    replication     replicator      10.128.0.17/32          md5
```

2) Reload Postgres (inside container):
```bash
cd /root
docker compose exec -u postgres postgres pg_ctl -D "$PGDATA" reload
```

3) Create slot for DC:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_create_physical_replication_slot('dc_slot');"
```

### Step 4.2 On DC: keep all Alfresco apps OFF, rebuild postgres data from DR
On **DC**:
```bash
cd /root
docker compose stop postgres
docker compose stop proxy content-app share alfresco transform-core-aio ocr-transformer solr6 activemq
```

Move old DB aside (safe rollback):
```bash
cd /root
mv ./volumes/data/postgres-data ./volumes/data/postgres-data_old_$(date +%F_%H%M%S)
mkdir -p ./volumes/data/postgres-data
chown -R 999:999 ./volumes/data/postgres-data
chmod 700 ./volumes/data/postgres-data
# If SELinux enforcing and you use :Z
chcon -Rt svirt_sandbox_file_t ./volumes/data/postgres-data || true
```

Run basebackup from DR → DC (makes standby automatically with `-R`):
```bash
cd /root
docker compose run --rm -u postgres --entrypoint bash postgres -lc '
export PGPASSWORD="StrongPass@123"
pg_basebackup -h 10.128.0.18 -U replicator -D "$PGDATA" -R -X stream -P --slot=dc_slot
'
```

Start DC postgres:
```bash
cd /root
docker compose up -d postgres
```

### Step 4.3 Verify DR ➜ DC replication
On **DC**:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_is_in_recovery();"
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT status, latest_end_lsn, latest_end_time FROM pg_stat_wal_receiver;"
```
Expect: `t`, `streaming`

On **DR**:
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT client_addr, state, sync_state, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;"
```

---

## 5) Contentstore Sync AFTER Failover (Primary=DR ➜ Standby=DC)

### Correct approach (recommended)
- Run rsync **ONLY on DC (standby)** pulling from DR (primary).
- NEVER run contentstore rsync on DR that pulls from DC.

### DC cron example (pull every minute)
On **DC**:
```bash
crontab -e
```
Add:
```cron
*/1 * * * * /usr/local/bin/alf-contentstore-sync.sh
```

**alf-contentstore-sync.sh** (DR ➜ DC, includes deletes):
```bash
#!/usr/bin/env bash
set -euo pipefail

DR_IP="10.128.0.18"
SSH_USER="drsync"
SSH_KEY="/root/.ssh/id_ed25519"
SSH_OPTS="-i ${SSH_KEY} -o StrictHostKeyChecking=accept-new"

SRC_BASE="${SSH_USER}@${DR_IP}:/root/volumes/data/alf-repo-data"
DST_BASE="/root/volumes/data/alf-repo-data"

LOG_DIR="/root/scripts/logs"
LOG="${LOG_DIR}/alf-contentstore-sync.log"
LOCK="/var/lock/alf-contentstore-sync.lock"

mkdir -p "$LOG_DIR" "$(dirname "$LOCK")"
exec 200>"$LOCK"
flock -n 200 || exit 0

ts(){ date -Is; }
echo "[$(ts)] START contentstore sync (DR -> DC)" >>"$LOG"

RSYNC_OPTS=(-aHAX --numeric-ids --delete --inplace --partial --info=progress2)

rsync "${RSYNC_OPTS[@]}" -e "ssh ${SSH_OPTS}"   "${SRC_BASE}/contentstore/"   "${DST_BASE}/contentstore/" >>"$LOG" 2>&1

rsync "${RSYNC_OPTS[@]}" -e "ssh ${SSH_OPTS}"   "${SRC_BASE}/contentstore.deleted/"   "${DST_BASE}/contentstore.deleted/" >>"$LOG" 2>&1

echo "[$(ts)] END contentstore sync" >>"$LOG"
```

---

## 6) Failback Drill (DR Primary ➜ DC Primary)
When you want to move production back to DC later:
1) Freeze writes on DR (stop Alfresco stack on DR)
2) Wait for DC standby catch-up (lag 0)
3) Final contentstore sync (DR ➜ DC)
4) Fence DR (stop DR postgres)
5) Promote DC (`pg_promote`)
6) Start full Alfresco stack on DC
7) Rebuild DR as standby from DC using `pg_basebackup` + slot `dr_slot`
8) Move rsync cron to the new standby (DR pulls from DC)

---

## 7) Common Troubleshooting

### 7.1 Preview/download fails with 404 + “Unable to locate content… cm:content”
Cause: DB row exists, but physical file missing in `contentstore/`.
Most common reason in this setup: **wrong-direction rsync with --delete** overwrote primary contentstore.
Fix:
- Stop that cron immediately on primary.
- Ensure only standby pulls from primary.
- Restore contentstore from backup if needed.

### 7.2 On standby: query errors about missing columns in pg_stat_wal_receiver
Use Postgres 16 compatible columns:
```sql
SELECT status, conninfo, receive_start_lsn, written_lsn, flushed_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_wal_receiver;
```

### 7.3 No rows in pg_stat_replication on primary
- pg_hba missing/incorrect
- slot missing
- standby not started or wrong conninfo
- firewall/network issue
