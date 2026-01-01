# DR setup for Docker
## Step 1: First create SSH key on DR server
```
ssh-keygen -t ed25519
```
## Step 2: Setup User In DC server

We create user name drsync  
```
sudo useradd -m -s /bin/bash drsync 2>/dev/null || true
sudo passwd -l drsync

sudo mkdir -p /home/drsync/.ssh
sudo chmod 700 /home/drsync/.ssh
sudo touch /home/drsync/.ssh/authorized_keys
sudo chmod 600 /home/drsync/.ssh/authorized_keys
sudo chown -R drsync:drsync /home/drsync/.ssh
```
## Step 3: Paste DR ssh key to DC server
```
echo 'ssh-key' \
| sudo tee -a /home/drsync/.ssh/authorized_keys >/dev/null
sudo chown drsync:drsync /home/drsync/.ssh/authorized_keys
sudo chmod 600 /home/drsync/.ssh/authorized_keys
sudo systemctl restart sshd
```
### Give drsync read-only access to contentstore
```
sudo setfacl -R -m u:drsync:rx ./volumes/data/alf-repo-data/contentstore
sudo setfacl -R -d -m u:drsync:rx ./volumes/data/alf-repo-data/contentstore

sudo setfacl -R -m u:drsync:rx ./volumes/data/alf-repo-data/contentstore.deleted 2>/dev/null || true
sudo setfacl -R -d -m u:drsync:rx ./volumes/data/alf-repo-data/contentstore.deleted 2>/dev/null || true
```
#### Ensure the remote Permission
```
chmod 700 ~/home/drsync/.ssh
chmod 600 ~/home/drsync/.ssh/authorized_keys
```
#### Test the connectivity from DR Server
```
ssh -i /home/ketan_gcloud/.ssh/id_ed25519 drsync@10.128.0.13 "whoami; hostname"
```
PostgreSQL Streaming Replication Setup (Docker Compose Alfresco)
================================================================

Overview
--------

*   **DC (Primary)**: Accepts writes from Alfresco.
*   **DR (Standby)**: Receives WAL continuously from DC using **physical streaming replication**.
*   DR database stays **read-only** while in recovery (`pg_is_in_recovery() = true`).
*   We used a **physical replication slot** `dr_slot` so WAL isn’t removed before DR receives it.

* * *

A) DC (Primary) PostgreSQL Configuration
========================================

A1) Load environment variables
------------------------------

Your `.env` contains:

*   `POSTGRES_USER=alfresco`
*   `POSTGRES_DB=alfresco`

On DC:

```bash
set -a
source .env
set +a
```

**Why:** Without this, `psql -U "$POSTGRES_USER"` becomes empty and postgres tries user `root` → you saw error:  
`FATAL: role "root" does not exist`.

* * *

A2) Create replication user (replicator)
----------------------------------------

On DC:

```bash
docker compose exec -T postgres psql -U alfresco -d postgres -c "
DO \$\$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='replicator') THEN
    CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'StrongPass@123';
  END IF;
END
\$\$;"
```

**Purpose:** Dedicated replication login with REPLICATION privilege.

* * *

A3) Enable required replication parameters (ALTER SYSTEM)
---------------------------------------------------------

On DC:

```bash
docker compose exec -T postgres psql -U alfresco -d postgres -c "
ALTER SYSTEM SET listen_addresses='*';
ALTER SYSTEM SET wal_level='replica';
ALTER SYSTEM SET max_wal_senders='10';
ALTER SYSTEM SET max_replication_slots='10';
ALTER SYSTEM SET wal_keep_size='2048MB';
"
docker compose restart postgres
```

**Purpose:**

*   `wal_level=replica`: enables WAL needed for replicas
*   `max_wal_senders`: allows standby connections
*   `replication_slots`: allow slots
*   `wal_keep_size`: keeps enough WAL (extra safety)
*   `listen_addresses='*'`: allow DR to connect

* * *

A4) Allow DR host in `pg_hba.conf`
----------------------------------

On DC we appended a line to the postgres `pg_hba.conf` (the file path was discovered using `show hba_file`).

Command used:

```bash
DR_IP="10.160.0.3"

docker compose exec -T postgres bash -lc '
HBA=$(psql -U "$POSTGRES_USER" -Atc "show hba_file");
echo "host replication replicator '"$DR_IP"'/32 md5" >> "$HBA";
tail -n 10 "$HBA"
'
docker compose exec -T postgres psql -U alfresco -d postgres -c "select pg_reload_conf();"
```

**Purpose:** permit DR server IP to connect for replication.

> Note: It got added twice once; we kept one correct line eventually.

* * *

A5) Create physical replication slot
------------------------------------

On DC:

```bash
docker compose exec -T postgres psql -U alfresco -d postgres -c \
"SELECT * FROM pg_create_physical_replication_slot('dr_slot');"
```

**Purpose:** ensures WAL is retained until DR consumes it.

* * *

A6) Primary-side verification
-----------------------------

On DC:

```bash
docker compose exec -T postgres psql -U alfresco -d postgres -c \
"select client_addr,state,sync_state,write_lag,flush_lag,replay_lag from pg_stat_replication;"
```

Expected:

*   `client_addr = 10.160.0.3`
*   `state = streaming`

* * *

B) DR (Standby) PostgreSQL Configuration
========================================

B1) Prepare standby with base backup from DC (critical step)
------------------------------------------------------------

We stopped DR postgres and ran `pg_basebackup` **from DR** to pull the database from DC.

Command (run on DR):

```bash
set -a; source .env; set +a
PRIMARY_IP="10.128.0.4" #DC sever ip
export REPL_PASSWORD='StrongPass@123'

docker compose stop postgres

docker compose run --rm \
  -e PGPASSWORD="$REPL_PASSWORD" \
  postgres bash -lc "
set -e
rm -rf \"\$PGDATA\"/*
pg_basebackup -h $PRIMARY_IP -p 5432 -U replicator -D \"\$PGDATA\" -Fp -Xs -P -R -S dr_slot
echo \"primary_slot_name = 'dr_slot'\" >> \"\$PGDATA/postgresql.auto.conf\"
"
docker compose up -d postgres
```

**What this did:**

*   `rm -rf "$PGDATA"/*` → cleans old standby data
*   `pg_basebackup`:
    *   copies a consistent base backup from DC
    *   `-R` writes standby config automatically (creates `standby.signal` + `primary_conninfo`)
    *   `-S dr_slot` uses the DC replication slot
*   `primary_slot_name='dr_slot'` ensures standby uses the slot.

* * *

B2) Standby-side verification (DR)
----------------------------------

On DR:

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c "select pg_is_in_recovery();"
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c "select * from pg_stat_wal_receiver;"
```

Expected:

*   `pg_is_in_recovery()` = `t`
*   `pg_stat_wal_receiver.status` = `streaming`
*   `sender_host` = `10.128.0.4`
*   `slot_name` = `dr_slot`

* * *

C) Common Issues We Hit (and Fixes)
===================================

C1) `FATAL: role "root" does not exist`
---------------------------------------

Cause: `.env` not loaded → `$POSTGRES_USER` empty.

Fix:

```bash
set -a; source .env; set +a
```

* * *

C2) DR query column mismatch (`received_lsn does not exist`)
------------------------------------------------------------

Cause: PostgreSQL version differences.  
Fix: Use columns that exist in your PG16 view:

*   `written_lsn`, `flushed_lsn`, `latest_end_lsn`.

* * *

D) Final Health Check Commands (for documentation)
==================================================

DC checks
---------

```bash
set -a; source .env; set +a
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select client_addr,state,sync_state,write_lag,flush_lag,replay_lag from pg_stat_replication;"
```

DR checks
---------

```bash
set -a; source .env; set +a
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c "select pg_is_in_recovery();"
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c "select * from pg_stat_wal_receiver;"
```

* * *

If you want, paste your **DC** and **DR** `postgres:` service sections from docker-compose (just that block), and I’ll convert this into a polished “SOP” format (Prereqs → Commands → Expected Output → Rollback → Failover/Failback notes).



