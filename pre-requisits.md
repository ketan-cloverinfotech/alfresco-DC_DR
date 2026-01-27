## Must check following
- docker subnet ip
- directory ownership (./volumes)

## Verification on DB
### 1.Check role details (exists? replication privilege?)
```
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT rolname,
       rolcanlogin,
       rolreplication,
       rolvaliduntil
FROM pg_roles
WHERE rolname='replicator';
"
```
### 2. Verify replication slot(s) exist (DC or DR)
```
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT slot_name,
       slot_type,
       active,
       restart_lsn,
       wal_status
FROM pg_replication_slots
ORDER BY slot_name;
"
```

### 3. WAL receiver status (Postgres 16)
```
cd /root
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT status,
       sender_host,
       sender_port,
       receive_start_lsn,
       written_lsn,
       flushed_lsn,
       latest_end_lsn,
       latest_end_time
FROM pg_stat_wal_receiver;
"
```
