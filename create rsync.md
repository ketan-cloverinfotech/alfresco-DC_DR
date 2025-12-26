# Transferred data using rsync

## Transfer content store folder
```
sudo rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2 \
  -e "ssh -i /home/ketan_gcloud/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new" \
  drsync@10.128.0.4:/home/ketan_gcloud/volumes/data/alf-repo-data/contentstore/ \
  /home/ketan_gcloud/volumes/data/alf-repo-data/contentstore/

```
## Transfer contentstore.deleted
```
sudo rsync -aHAX --numeric-ids --delete --inplace --partial --info=progress2 \
  -e "ssh -i /home/ketan_gcloud/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new" \
  drsync@10.128.0.4:/home/ketan_gcloud/volumes/data/alf-repo-data/contentstore.deleted/ \
  /home/ketan_gcloud/volumes/data/alf-repo-data/contentstore.deleted/
```
