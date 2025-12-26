# DR setup for Docker

**User:** Ketan Thombare (ketan.thombare01@gmail.com)  
**Created:** 12/26/2025 10:05:30  
**Updated:** 12/26/2025 18:53:51  
**Exported:** 12/26/2025 18:58:04  
**Link:** [https://chatgpt.com/g/g-p-694be3c5266481918c1e73da54021e52/c/694e108f-0e30-8323-a2f8-7afb3d3e4ad8](https://chatgpt.com/g/g-p-694be3c5266481918c1e73da54021e52/c/694e108f-0e30-8323-a2f8-7afb3d3e4ad8)  

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