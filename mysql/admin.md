---
title: mysql/admin
---

- create dump 2 diff flavours
  - somewhat fancy dump omitting mysql database and schema dbs for creating slave

    ```sh
    mysqldump \
      --verbose \
      --skip-lock-tables \
      --quick \
      --skip-set-charset \
      --default-character-set=utf8 \
      --single-transaction \
      --flush-logs \
      --routines \
      --master-data=2 \
      --databases $(echo "show databases" | mysql | grep -Ev "^(Database|mysql|performance_schema|information_schema)$") \
      | pv > dump.sql
    ```

  - quick and dirty

    ```sh
    mysqldump \
      --verbose \
      --quick \
      --skip-lock-tables \
      --single-transaction \
      --insert-ignore
    ```

- database too large for mysql dump on db server (dump for creating slave)
  - use backup server or something like that
    - create temp user (if no backup or root from network allowed)
      - check if such users exist

        ```sql
        mysql> select host,user from mysql.user where user regexp 'root|backup';
        +-----------+------+
        | host      | user |
        +-----------+------+
        | 127.0.0.1 | root |
        | localhost | root |
        +-----------+------+
        ```

      - create user if doesnt exist with grants

        ```sql
        mysql> create user 'temp'@'%' identified by 'temp';
        Query OK, 0 rows affected (0.00 sec)
        
        mysql> select host,user from mysql.user where user regexp 'root|backup|temp';
        +-----------+------+
        | host      | user |
        +-----------+------+
        | %         | temp |
        | 127.0.0.1 | root |
        | localhost | root |
        +-----------+------+
        3 rows in set (0.00 sec)
        
        mysql> grant all on *.* to 'temp'@'%';
        ```

    - create defaults files on backup server

      ```sh
      mkdir /backup/temp
      vi /backup/temp/db-master2.example.internal
      [client]
      user=temp
      password="temp"
      host=db-master2.example.internal
      [mysqldump]
      user=temp
      password="temp"
      host=db-master2.example.internal
      ```

    - create dump using defaults file

      ```sh
      screen -S db-master2-dump
      mysqldump \
        --defaults-file=/backup/temp/db-master2.example.internal \
        --verbose \
        --skip-lock-tables \
        --quick \
        --skip-set-charset \
        --single-transaction \
        --flush-logs \
        --routines \
        --master-data=2 \
        --databases $(echo "show databases" | mysql --defaults-file=/backup/temp/db-master2.example.internal | grep -Ev "^(Database|performance_schema|information_schema)$") \
        | pv > /backup/temp/db-master2.sql
      ```

    - source dump on slave (assuming you created defaults file)
      - defaults file

        ```sh
        cat db-slave1.example.internal 
        [client]
        user=temp
        password="temp"
        host=db-slave1.example.internal
        ```

      - use mysql console to source dump

          ```sh
          screen -S db-slave1-restore
          mysql --defaults-file=/backup/temp/db-slave1.example.internal
          ```

          ```mysql
          mysql> tee log.out
          Logging to file 'log.out'
          mysql> source db-master2.sql
          # exit screen
          ```

  - use master data from dump to set up slave

    ```sh
    mysql --defaults-file=/backup/temp/db-master2.example.internal
    ```

    ```mysql
    mysql> change master to
        MASTER_LOG_FILE='<new_master_log_file>',
        MASTER_LOG_POS='<new_master_log_pos>';
    mysql> start slave;
    ```

- read binlog

  ```sh
  mysqlbinlog binlog.032198 --start-position=878837181 --base64-output=DECODE-ROWS --verbose
  ```
