---
title: mysql/binlog-size
---

- sum binlog sizes in GiB (mysql + mariadb)

  ```sh
  echo $(($(ls -l mysql-bin.* | awk '{print $5}' | paste -s -d "+" | bc)/2**30))
  ```
