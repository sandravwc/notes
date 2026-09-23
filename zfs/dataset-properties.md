---
title: zfs/dataset-properties
---

- read the properties that matter across every dataset of a pool

  ```sh
  a='zfs get compression,recordsize,atime,sync,primarycache,secondarycache'
  for pool in $(zfs list | awk '/zmysql/ {print $1}'); do eval $a $pool; echo; done
  ```
