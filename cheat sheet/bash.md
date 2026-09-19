---
title: cheat sheet/bash
tags: [cheat sheet]
---

- parameter expansion search and replace
  ```sh
  my_var="2025/10/29"
  $ echo "${my_var//\//-}"
  2025-10-29
  
  $ file="report_2024_final.txt"
  $ echo ${file/2024/2025}
  report_2025_final.txt
  ```
- search and execute
  ```sh
  awk -F "]=" '/check_mysql/ {print $2}' /etc/nagios/nrpe_local.cfg | xargs -I % sh -c "echo %; %; echo \$?; echo"
  ```
