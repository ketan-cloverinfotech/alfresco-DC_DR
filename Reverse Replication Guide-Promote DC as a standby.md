# Promote DC as A standby

## Step 0: Pre-requisits
- Stop all cronjobs
- check DC is standy or not step 3.1
- 


Below are the **exact commands**.

* * *

## Step 1: On DR (10.128.0.18) — allow DC to connect for replication

### 1.1 Add DC replication rule in DR `pg_hba.conf`

On **DR host**, edit your mapped file:

```bash
vi /root/volumes/config/postgres/pg_hba.conf
```

Add this line (use same auth method you used before; your DC used `md5`, so keep `md5`):

```conf
# DC (10.128.0.17) connects to DR primary as user "replicator"
host    replication     replicator      10.128.0.17/32          md5
```

### 1.2 Reload postgres on DR

```bash
cd /root
docker compose exec -u postgres postgres pg_ctl -D "$PGDATA" reload
```

### 1.3 (Recommended) Create a new replication slot on DR for DC
On **DR**
```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_create_physical_replication_slot('dc_slot');"
```

> If it says “already exists”, that’s fine.

* * *

2) On DC (10.128.0.17) — rebuild postgres data as standby from DR
=================================================================

Step:2 Stop postgres on DC (and keep apps stopped)
-----------------------------------------------

```bash
cd /root
docker compose stop postgres
# (leave alfresco stack stopped on DC while DR is active)
```

2.2 Move old DC database aside (DESTRUCTIVE but safe)
-----------------------------------------------------

```bash
cd /root
mv ./volumes/data/postgres-data ./volumes/data/postgres-data_old_$(date +%F_%H%M%S)
mkdir -p ./volumes/data/postgres-data
chown -R 999:999 ./volumes/data/postgres-data
chmod 700 ./volumes/data/postgres-data
```

### If SELinux is enforcing (because you use `:Z`)

Run this once on DC:

```bash
chcon -Rt svirt_sandbox_file_t ./volumes/data/postgres-data
```

2.3 Take base backup from DR → DC (this makes DC standby)
---------------------------------------------------------

Run this on **DC**:

```bash
cd /root
docker compose run --rm -u postgres --entrypoint bash postgres -lc '
export PGPASSWORD="StrongPass@123"
pg_basebackup -h 10.128.0.18 -U replicator -D "$PGDATA" -R -X stream -P --slot=dc_slot
'
```

What this does:

*   Copies DR primary database to DC
*   `-R` writes standby config automatically (`standby.signal` + `primary_conninfo`)
*   Uses replication slot `dc_slot`

2.4 Start postgres on DC (now standby)
--------------------------------------

```bash
cd /root
docker compose up -d postgres
```

* * *

3) Verify replication is now DR(primary) ➜ DC(standby)
======================================================

3.1 On DC: confirm it is standby
--------------------------------

```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_is_in_recovery();"
```

Expected: `t`

Also check WAL receiver is running:

```bash
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT status, conninfo FROM pg_stat_wal_receiver;"
```

3.2 On DR: confirm DC is connected as replica
---------------------------------------------

```bash
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

Expected: you see `10.128.0.17` in `client_addr`.

* * *

Important rule now
==================

✅ Keep **DC Alfresco stack OFF** while DR is primary, otherwise you risk split-brain at the application/data level.

* * *

If you want, paste the output of these two commands (so I can confirm it’s perfect):

**On DC**

```bash
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT status, conninfo FROM pg_stat_wal_receiver;"
```

**On DR**

```bash
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```




