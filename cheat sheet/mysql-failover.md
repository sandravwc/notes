---
title: cheat sheet/mysql-failover
---

- mysql master/slave failover behind keepalived

  - ```sh
    cp /etc/keepalived/conf/db-pair.conf /etc/keepalived/conf/db-pair.conf.failover
    vi /etc/keepalived/conf/db-pair.conf.failover   # swap real ips
    sed -i '/db-pair.conf/s/^/#/' /etc/keepalived/keepalived.conf
    systemctl reload keepalived
    awk '/db-pair/ {print $2}' /etc/motd | xargs -I % ssh -n % "systemctl stop mysqld"
    awk '/db-pair/ {print $2}' /etc/motd | xargs -I % ssh -n % "zfs snapshot -r zmysql/main@$(date +%s)"
    # swap dns cnames for the pair here
    awk '/db-pair/ {print $2}' /etc/motd | xargs -I % ssh -n % "systemctl start mysqld"
    mysql --execute "stop slave; show master status;"      # new master, note file+pos
    change master to master_host="<ip>",master_user="repl",master_password="<redacted>",master_log_file="mysql-bin.008877",master_log_pos=453320903;   -- new slave
    mysql --execute "reset slave all"                       # new master
    nmcli con mod dummy0 ipv4.addresses "<ip>/32"; nmcli con up dummy0   # both, swapped roles
    cp /etc/keepalived/conf/db-pair.conf.failover /etc/keepalived/conf/db-pair.conf
    sed -i '/db-pair.conf/s/^#//' /etc/keepalived/keepalived.conf
    systemctl reload keepalived
    ```
