# DR setup for Docker

**User:** Ketan Thombare (ketan.thombare01@gmail.com)  
**Created:** 12/26/2025 10:05:30  
**Updated:** 12/26/2025 18:53:51  
**Exported:** 12/26/2025 18:59:31  
**Link:** [https://chatgpt.com/g/g-p-694be3c5266481918c1e73da54021e52/c/694e108f-0e30-8323-a2f8-7afb3d3e4ad8](https://chatgpt.com/g/g-p-694be3c5266481918c1e73da54021e52/c/694e108f-0e30-8323-a2f8-7afb3d3e4ad8)  

## Response:
Below is the **exact solution** we used for **Postgres DR (streaming + failover)** and the **exact solution** for **rsync (Alfresco contentstore sync)** — with the _why_, _commands_, and _validation_.

* * *

A) Postgres solution (DC = Primary, DR = Standby)
=================================================

A1) DC (Primary) — enable streaming replication
-----------------------------------------------

### 1) Create replication user (inside DC postgres container)

```bash
set -a; source .env; set +a

docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"DO \$\$
BEGIN
IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='replicator') THEN
  CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'StrongPass@123';
END IF;
END \$\$;"
```

### 2) Set required Postgres parameters (wal\_level etc.)

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"ALTER SYSTEM SET listen_addresses='*';
 ALTER SYSTEM SET wal_level='replica';
 ALTER SYSTEM SET max_wal_senders='10';
 ALTER SYSTEM SET max_replication_slots='10';
 ALTER SYSTEM SET wal_keep_size='2048MB';
 ALTER SYSTEM SET hot_standby='on';"

docker compose restart postgres
```

Validate:

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"show listen_addresses; show wal_level; show max_wal_senders; show max_replication_slots; show wal_keep_size;"
```

### 3) Allow DR IP in `pg_hba.conf` (THIS FIXES your “no pg\_hba.conf entry” error)

On DC host:

```bash
DR_IP="10.160.0.14"   # DR server IP

docker compose exec -T postgres bash -lc '
HBA=$(psql -U "$POSTGRES_USER" -Atc "show hba_file");
echo "host replication replicator '"$DR_IP"'/32 md5" >> "$HBA";
tail -n 10 "$HBA"
'

docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c "select pg_reload_conf();"
```

### 4) Create replication slot on DC

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"SELECT * FROM pg_create_physical_replication_slot('dr_slot');"
```

Validate DC sees standby connection (after DR starts):

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select client_addr,state,sync_state,write_lag,flush_lag,replay_lag from pg_stat_replication;"
```

* * *

A2) DR (Standby) — take basebackup + start streaming
----------------------------------------------------

### 1) Stop postgres on DR

```bash
cd ~/   # wherever docker-compose.yaml is
docker compose stop postgres
```

### 2) Rebuild PGDATA from Primary using `pg_basebackup`

```bash
PRIMARY_IP="10.128.0.13"         # DC server IP
export REPL_PASSWORD='StrongPass@123'

docker compose run --rm \
  -e PGPASSWORD="$REPL_PASSWORD" \
  postgres bash -lc "
set -e
rm -rf \"\$PGDATA\"/*
pg_basebackup -h $PRIMARY_IP -p 5432 -U replicator -D \"\$PGDATA\" -Fp -Xs -P -R -S dr_slot
echo \"primary_slot_name = 'dr_slot'\" >> \"\$PGDATA/postgresql.auto.conf\"
"
```

### 3) Start postgres on DR

```bash
docker compose up -d postgres
```

### 4) Validate DR is streaming standby

```bash
set -a; source .env; set +a

docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select pg_is_in_recovery();"

docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select status, written_lsn, flushed_lsn, latest_end_lsn, last_msg_receipt_time from pg_stat_wal_receiver;"
```

**Expected on standby:**

*   `pg_is_in_recovery = t`
*   `pg_stat_wal_receiver.status = streaming`

* * *

A3) DR Failover (when you want DR to become Active)
---------------------------------------------------

When DC is down / planned failover:

### 1) Stop Alfresco services first (important)

```bash
docker compose stop alfresco share content-app proxy solr6 transform-core-aio ocr-transformer activemq
```

### 2) Promote Postgres on DR

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select pg_promote(wait_seconds => 60);"
```

Validate:

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c \
"select pg_is_in_recovery(); show transaction_read_only;"
```

**Expected after promote:**

*   `pg_is_in_recovery = f`
*   `transaction_read_only = off`

### 3) Start whole Alfresco stack on DR

```bash
docker compose up -d
```

* * *

B) rsync solution (Alfresco contentstore sync DC → DR)
======================================================

The real problem you hit
------------------------

Your rsync failed because:

*   You used: `--rsync-path="sudo rsync"`
*   On DC, `sudo` asked for a password / needed tty:  
    `sudo: a terminal is required ...`

So contentstore never fully synced → Alfresco couldn’t find `.bin` files → Share got 500.

* * *

B1) Correct way (recommended): use a dedicated sync user + passwordless sudo for rsync (DC side)
------------------------------------------------------------------------------------------------

### 1) On DC: create user (example `drsync`)

```bash
sudo useradd -m -s /bin/bash drsync
sudo passwd -l drsync
sudo mkdir -p /home/drsync/.ssh
sudo chmod 700 /home/drsync/.ssh
sudo chown -R drsync:drsync /home/drsync/.ssh
```

### 2) On DC: allow that user to run rsync as root without password

Create sudoers file:

```bash
sudo visudo -f /etc/sudoers.d/drsync-rsync
```

Put:

```text
drsync ALL=(root) NOPASSWD: /usr/bin/rsync
```

Then:

```bash
sudo chmod 440 /etc/sudoers.d/drsync-rsync
```

✅ Now `--rsync-path="sudo rsync"` will work in non-interactive mode.

* * *

B2) Correct rsync command (run on DR)
-------------------------------------

### Full alf\_data sync (contentstore + deleted + indexes etc.)

```bash
sudo rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2 \
  --rsync-path="sudo rsync" \
  -e "ssh -i /home/ketan_gcloud/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new" \
  drsync@10.128.0.13:/home/ketan_gcloud/volumes/data/alf-repo-data/ \
  /home/ketan_gcloud/volumes/data/alf-repo-data/
```

### If you want only contentstore (faster)

```bash
sudo rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2 \
  --rsync-path="sudo rsync" \
  -e "ssh -i /home/ketan_gcloud/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new" \
  drsync@10.128.0.13:/home/ketan_gcloud/volumes/data/alf-repo-data/contentstore/ \
  /home/ketan_gcloud/volumes/data/alf-repo-data/contentstore/
```

* * *

B3) Why this fixes the 500 error
--------------------------------

That 500 happened because DB pointed to content URLs like:

`store://2025/12/26/.../file.bin`

But the `.bin` file didn’t exist on DR disk.

Once rsync correctly copied those missing `.bin` files (with correct UID/GID because of `--numeric-ids`), Alfresco could read the content → Share stopped failing.

* * *

Quick validation checklist (must pass)
======================================

1) On DR (standby mode)
-----------------------

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select pg_is_in_recovery();"
```

Expect: `t`

2) On DC (primary) sees DR
--------------------------

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d postgres -c \
"select client_addr,state,sync_state from pg_stat_replication;"
```

Expect: row with DR IP + streaming

3) Verify a content URL exists as a file on DR
----------------------------------------------

From DB:

```bash
UUID="xxxx-xxxx..."
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Atc "
SELECT cu.content_url
FROM alf_node n
JOIN alf_node_properties np ON np.node_id=n.id
JOIN alf_qname q ON np.qname_id=q.id
JOIN alf_namespace ns ON q.ns_id=ns.id
JOIN alf_content_data cd ON cd.id=np.long_value
JOIN alf_content_url cu ON cu.id=cd.content_url_id
WHERE n.uuid='${UUID}'
AND ns.uri='http://www.alfresco.org/model/content/1.0'
AND q.local_name='content';
"
```

Then check file exists:

```bash
sudo ls -l /home/ketan_gcloud/volumes/data/alf-repo-data/contentstore/YYYY/MM/DD/...
```

* * *

If you want, I can convert the above into a **clean DR SOP document** with sections:

*   DC initial setup
*   DR initial setup
*   Daily sync procedure
*   Failover procedure
*   Failback procedure
*   Troubleshooting (the exact 500 error scenario)

## Response:
Not **all** physical data. Only the parts that must be rebuilt.

1) Postgres DR (standby rebuild)
--------------------------------

✅ **Yes — you must wipe DR PGDATA** _only when you are re-initializing standby_ (running `pg_basebackup` again), otherwise `pg_basebackup` will fail or you’ll keep old timeline/cluster state.

On DR:

```bash
docker compose stop postgres
sudo rm -rf ./volumes/data/postgres-data/*
# or inside container PGDATA, but host path is easiest
```

Then run `pg_basebackup` again.

**Don’t delete Postgres data** if:

*   DR is already streaming fine, and
*   you’re not changing primary, slot, or rebuilding standby.

* * *

2) Alfresco content (alf\_data / contentstore)
----------------------------------------------

❌ **No — do NOT delete DR alf-repo-data** in normal DR sync.

For Alfresco content, the whole point is to keep DR contentstore continuously updated via rsync.

You only delete Alfresco alf\_data in special cases like:

*   you copied wrong data to the wrong path,
*   permissions/SELinux labels are totally broken and faster to re-seed clean,
*   you want a one-time fresh full copy from DC (then you can delete and do a fresh rsync).

Even then, usually you **don’t need to delete**—just do rsync with `--delete` to converge.

* * *

3) Solr index data (search)
---------------------------

⚠️ Optional, depends on your DR approach.

In DR, Solr data can be rebuilt from DB/content, so many teams do:

*   **delete DR solr-data** and let it reindex after failover.

Path (from your compose):

```bash
sudo rm -rf ./volumes/data/solr-data/*
```

This is not required to fix DB/content consistency, only search/index consistency.

* * *

Practical rule
==============

### If you are doing a “fresh rebuild”:

*   ✅ Delete **postgres-data** on DR (mandatory for new basebackup)
*   ✅ Keep **alf-repo-data** (recommended), or rsync `--delete` from DC
*   ✅ Optionally delete **solr-data** (reindex later)

### If you are doing normal ongoing DR sync:

*   ❌ Don’t delete anything
*   ✅ Just keep:
    *   Postgres streaming replication
    *   rsync contentstore periodically (or continuously)

If you tell me whether this DR is **hot standby only** (no users) or **active-passive failover**, I’ll give the exact “safe sequence” you should follow every time.



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)