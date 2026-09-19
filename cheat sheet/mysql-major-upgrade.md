---
title: cheat sheet/mysql-major-upgrade
tags: [cheat sheet]
---

- percona mysql upgrade via pcs (5.7 -> 8.0 -> 8.4lts)
  - pre upgrade
    - ```sh
      zfs snapshot -r zmysql/main@before_mysqld_upgrade$(date +%s)   # master
      pcs resource update r_mysqlserver op start timeout=1200s
      varnishadm backend.set_health 'web0[12345]-vm' sick
      pcs resource update r_mysqlserver additional_parameters="--bind-address=127.0.0.1"
      ```
  - upgrade to 80
    - ```sh
      # replica
      stop slave;
      # master
      pcs resource disable r_mysqlserver
      percona-release setup ps80
      dnf remove Percona-Server-server-57 --assumeyes
      dnf swap Percona-Server-client-57 percona-server-client --assumeyes
      dnf install percona-server-server --assumeyes
      mv /etc/my.cnf_backup-* /etc/my.cnf
      pcs resource enable r_mysqlserver
      # replica: same binary upgrade, systemctl stop/start mysqld instead of pcs
      ```
  - upgrade to 84lts
    - ```sh
      percona-release setup ps84lts
      dnf update percona-server-server percona-server-client --assumeyes
      # add mysql_native_password to /etc/my.cnf if still needed
      ```
  - post upgrade
    - ```sh
      show binary log status\G   -- note position in case replication breaks
      pcs resource update r_mysqlserver additional_parameters="--bind-address=0.0.0.0"
      start replica; show replica status;
      varnishadm backend.set_health 'web0[12345]-vm' auto
      zfs destroy zmysql/main@before_mysqld_upgrade...
      pcs resource update r_mysqlserver op start timeout=60s
      ```
