---
title: mysql/admin-queries
---

- show db data usage as seen by sql

  ```sql
  mysql> select table_schema, count(*), sum(table_rows), sum(data_length)/pow(2, 30) data_gb from information_schema.TABLES where table_schema like 'foo%' group by 1;
  +--------------+----------+-----------------+-------------------+
  | table_schema | count(*) | sum(table_rows) | data_gb           |
  +--------------+----------+-----------------+-------------------+
  | foo          |      337 |      4146714152 | 381.5830068765208 |
  | foo_bar      |      613 |       351882429 | 65.07333374023438 |
  +--------------+----------+-----------------+-------------------+
  2 rows in set (0,54 sec)
  ```

- optimize tables based on select condition

  ```sql
  use mysql;
  delimiter $$
  create procedure OptimizeAllCompactTables()
  begin
    declare finished integer default 0;
    declare tbl_schema varchar(9999);
    declare tbl_name varchar(9999);
    declare cur cursor for
      select table_schema, table_name 
      from information_schema.tables 
      where row_format regexp 'Compact';
    declare continue handler for not found set finished = 1;
    open cur;
    loop_tables: loop
      fetch cur into tbl_schema, tbl_name;
      if finished then 
        leave loop_tables;
      end if;
      set @stmt = concat('optimize table ', tbl_schema, '.', tbl_name);
      select @stmt;
      prepare stmt from @stmt;
      execute stmt;
      deallocate prepare stmt;
    end loop;
    close cur;
  end $$
  delimiter ;
  call OptimizeAllCompactTables();
  drop procedure if exists OptimizeAllCompactTables;
  ```
