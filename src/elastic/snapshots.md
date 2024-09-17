# Elastic backup receipes

To be able to backup elasticsearch indices first of all we have to define a repository whch could be either a local storage or a remote storage.

## Storage repositories
### Local snapshot repository 

To define local storage point we specify a repository path in  ```elasticsearch.yml``` as follows:
```yml
path.repo: ["/data/elastic-backup"]
```

and perform restart

Next step is to initialize storage repository in elastic by running 

```sh
curl -X PUT -H 'Content-Type: application/json' -d '{"type":"fs","settings":{"location":"/data/elastic-backup"}}' "http://<host>:9200/_snapshot/<repository_name>"
```

### Remote S3 repository

You can use any S3-compatible storage as a snaposhot repository. To be able to do this we need to install the repository-s3 plugin (for old versions):

Check if the plugin is installed
```sh
ES_PATH_CONF=/etc/elasticsearch bin/elasticsearch-plugin list
```

Install the plugin
```sh
bin/elasticsearch-plugin install repository-s3
```

For new versions the plugin is already installed so the next step is..

Define client settings (especially endpoint URL) in the ```elasticsearch.yml``` (consider not using the bucket name in the endpoin url)
```sh
s3.client.dospaces.endpoint: sfo2.digitaloceanspaces.com
s3.client.dospaces.protocol: https
```

Add the endpoint credentials to the elasticsearch private storage
```
bin/elasticsearch-keystore add s3.client.dospaces.access_key
bin/elasticsearch-keystore add s3.client.dospaces.secret_key
```

Restart elasticsearch

Init s3 remote repository
```sh
curl -X PUT -H 'Content-Type: application/json' -k -d '{"type":"s3","settings":{"bucket":"<bucket_name>","client":"dospaces","base_path":"elastic-backup"}}' "http://<host>:9200/_snapshot/<repository_name>"
```

## Performing snapshot

```sh
curl -X PUT "http://<host>:9200/_snapshot/<repository_name>/<snapshot_name>"
```

wait for the status to be SUCCESS
```sh
curl -X GET "http://<host>:9200/_snapshot/<repository_name>/<snapshot_name>"
```

## Removing snapshot
```sh
curl -X DELETE "http://<host>:9200/_snapshot/<repository_name>/<snapshot_name>"
```

## Daily backup keeping last N backups

```sh
#!/bin/sh

CURL=`which curl`
TODAY=`date +"%d%m%Y"`
HOST="http://10.10.10.1:9200"
REPOSITORY="s3-backup"
BACKUP_LIMIT=3
ARGS="-u user:pass --insecure"

# Get current snapshots list and get the latest BACKUP_LIMIT entrys
$CURL -s -X GET $ARGS "${HOST}/_snapshot/${REPOSITORY}/_all" | jq '.snapshots[].snapshot' | sort -r | tr -d \" > snapshot.list
cat snapshot.list | head -$BACKUP_LIMIT > snapshot.keep

# For all the snapshot missing in to keep entrys - delete them
for snapshot in `grep -v -f snapshot.keep snapshot.list`; do
  $CURL -s -X GET $ARGS "${HOST}/_snapshot/${REPOSITORY}/${snapshot}" | jq -e '.snapshots[] | .state=="SUCCESS"' > /dev/null
  if [ $? -eq 0 ]; then
     $CURL -s -X DELETE $ARGS "${HOST}/_snapshot/${REPOSITORY}/${snapshot}"
  fi
done;

# We make today backup only if it doesn't exist in snapshots list
echo $TODAY | grep -vf snapshot.list
if [ $? -eq 0 ]; then
  $CURL -s -X PUT $ARGS "${HOST}/_snapshot/${REPOSITORY}/${TODAY}" > backup.output
fi
```

backup.output should be
```json
{"acknowledged":true}
```