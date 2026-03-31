# Elasticsearch: read-only indices and disk watermarks

When the cluster thinks a data node is low on disk, Elasticsearch can set indices to **read-only** via the `read_only_allow_delete` block. Writes fail until you fix disk pressure or clear the block. The steps below check that state, optionally relax cluster disk rules, and remove the block from indices.

## Check read-only settings on all indices

Returns whether each index has `blocks.read_only_allow_delete` set:

```bash
curl -k -u elastic:123456 -XGET \
  'https://localhost:9200/_all/_settings?filter_path=**.blocks.read_only_allow_delete'
```

## Temporarily disable cluster disk allocation thresholds

If nodes are borderline on free space, the allocator may keep enforcing read-only. You can turn off disk threshold checks **transiently** (restarts or persistent settings can override this):

```bash
curl -k -u elastic:123456 \
  -H 'Content-Type: application/json' \
  -X PUT 'https://localhost:9200/_cluster/settings' \
  -d '{"transient":{"cluster.routing.allocation.disk.threshold_enabled": false}}'
```

Prefer fixing disk usage (delete old indices, add capacity) over leaving thresholds disabled.

## Clear read-only marks on all indices

After disk is healthy (or thresholds adjusted), remove the block so writes work again:

```bash
curl -k -u elastic:123456 \
  -H 'Content-Type: application/json' \
  -X PUT 'https://localhost:9200/_all/_settings' \
  -d '{"index.blocks.read_only_allow_delete": null}'
```

Target a single index by replacing `_all` with the index name if you only need to fix one.
