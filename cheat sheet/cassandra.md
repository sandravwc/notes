---
title: cheat sheet/cassandra
---

- compaction per sstable

  ```sh
  nodetool -u cassandra-superuser -pw <redacted> compact --user-defined /var/lib/cassandra/data/my_keyspace/certificates-2b28efb0aea611ec8b916931b1205bf8/
  nodetool -u cassandra-superuser -pw <redacted> compactionhistory
  ```
