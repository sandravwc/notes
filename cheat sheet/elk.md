---
title: cheat sheet/elk
---

- moving indices between nodes

  ```sh
  # location of index non primary shards
  curl -s -u admin:<redacted> 192.168.1.10:9200/_cluster/allocation/explain?pretty -H 'Content-Type: application/json' -d'
  {
  "index": "zone_data_stats",
  "shard": 0,
  "primary": false
  }
  '
  # get location of index primary shard
  - curl -X GET "localhost:9200/_cluster/allocation/explain?pretty" -H 'Content-Type: application/json' -d'
  {
  "index": "zone_data_stats,
  "shard": 0,
  "primary": true
  }
  '
  # move index to different node
  - curl -X POST -u admin:<redacted> "192.168.1.10:9200/_cluster/reroute?metric=none&pretty" -H 'Content-Type: application/json' -d'
  {
  "commands": [
    {
      "move": {
        "index": "zone_data_stats", "shard": 0,
        "from_node": "elk06", "to_node": "elk04"
      }
    }
  ]
  }
  '
  # get status of current operation e.g. relocating index
  - curl -XPOST -u admin:<redacted> "192.168.1.10:9200/_cat/recovery?v=true"
  - [19:39:35][root@elk01:~]$ curl -s -XGET -u admin:<redacted> "localhost:9200/_cat/recovery/zone_data_stats?v=true"
  index           shard time  type           stage source_host     source_node     target_host     target_node     repository snapshot files files_recovered files_percent files_total bytes       bytes_recovered bytes_percent bytes_total translog_ops translog_ops_recovered translog_ops_percent
  zone_data_stats 0     24.4m peer           index 192.168.1.11     elk06           192.168.1.12    elk04           n/a        n/a      116   115             99.1%         116         10341330408 8268205727      80.0%         10341330408 0            0                      100.0%
  zone_data_stats 0     488ms existing_store done  n/a             n/a             192.168.1.11     elk06           n/a        n/a      0     0               100.0%        116         0           0               100.0%        10341330408 0            0                      100.0%
  ```

- mehr max shards

  ```sh
  curl -XPUT 'http://192.168.1.20:9200/_cluster/settings' -H 'Content-Type: application/json' -d'
  {
    "persistent" : {
      "cluster.routing.allocation.total_shards_per_node" : 3500
    }
  }'
  curl -XPUT 'http://192.168.1.20:9200/_cluster/settings' -H 'Content-Type: application/json' -d'
  {
    "persistent": {
      "cluster": {
        "max_shards_per_node": 3500
      }
    }
  }'
  ```

- index status

  ```sh
  curl -XGET http://192.168.1.20:9200/_cluster/allocation/explain -H 'Content-Type: application/json' -d'
  {
    "index": "my-index-2025.03.31",
    "shard": 0,
    "primary": true
  }' | jq
  ```

- retry failed shard allocation, and ask why it failed

  ```sh
  curl -XPOST 10.0.2.131:9200/_cluster/reroute?retry_failed=true
  curl -XPOST 10.0.2.131:9200/_cluster/allocation/explain
  ```
