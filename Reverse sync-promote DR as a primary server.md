# Reverse sync-promote DR as a primary server
## Step 1: Verify wether your DR is standby or not ( didnt accept data )
Note: there is no query for DR as it make as stand by the basebackup query make DR as standby
_to check whether it is standby or not first go your docker compose directory and run following query_
```
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "SELECT pg_is_in_recovery();"
```
if output get **f** then it is primary,if o/p get **t** it is standby in **our case on DR it should be t(standby)**

## Step 2: Freeze writes on DC (stop Alfresco stack on DC)
first go your docker compose directory and run following
```
docker compose stop proxy content-app share alfresco transform-core-aio ocr-transformer solr6 activemq
```
## Step 3: Wait until DR is fully caught up (important)
On **DC**
**Check DC replication lag**
```
docker compose exec -u postgres postgres psql -U alfresco -d postgres -c "
SELECT client_addr, state,
       sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;"
```
You should get following output(must)


_state = streaming
lag_bytes ideally 0 (or very small)_

