# MySQL Replication Playground

master 目录下的文件:

- mysqld.cnf, 配置文件.

- mysqlcli.sh, 启动 master 服务器容器中的 mysql 命令行.

- bash.sh, 启动 master 服务器容器中的 bash 命令行.

slave 目录下的文件和 master 目录中的文件类似.


## 设置全新的主从复制

启动两个全新的 MySQL 服务器, 一开始时它们都没有写入任何数据.

使用 docker-compose 来创建和管理服务器, 配置文件在这里[docker-compose.yml](./docker-compose.yml).

步骤:

- 使用 docker-compose 启动服务器.
  
  ```sh
  docker-compose up
  ```

- 在 master 服务器创建一个用户用于复制:

  ```sql
  CREATE USER 'repl'@'%' IDENTIFIED BY 'repl1234';
  GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
  ```

- 查看 master 服务器的 binlog 信息:

  ```sql
  mysql> show master status;
  +---------------+----------+--------------+------------------+-------------------+
  | File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
  +---------------+----------+--------------+------------------+-------------------+
  | master.000003 |      689 |              |                  |                   |
  +---------------+----------+--------------+------------------+-------------------+
  1 row in set (0.00 sec)
  ```

  记住 File 和 Position. 

- 开始复制, 在 slave 执行下面的 SQL:

  ```sql
  CHANGE MASTER TO
      MASTER_HOST = 'mysql_master',
      MASTER_USER = 'repl',
      MASTER_PASSWORD = 'repl1234',
      MASTER_LOG_FILE = 'master.000003',
      MASTER_LOG_POS = 689;

  START SLAVE;
  ```

  注意上面 SQL 中的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 是前面在 master 服务器上执行 `show master status` 的结果.

  从 slave 的日志可以看到下面的输出:

  ```
  2021-01-30T12:35:37.173915Z 9 [System] [MY-010562] [Repl] Slave I/O thread for channel '': connected to master 'repl@mysql_master:3306',replication started in log 'master.000003' at position 689
  ```

- 在 master 服务器插入数据:

  ```sql
  CREATE DATABASE test;
  USE test;
  CREATE TABLE test(id INT PRIMARY KEY AUTO_INCREMENT, a INT, b CHAR(10));
  INSERT INTO test(a, b) VALUES
      (1, 'hello'),
      (2, 'world'),
      (3, 'foo'),
      (4, 'bar');
  ```

- 在 slave 上查看复制的数据:

  ```sql
  mysql> select * from test;
  +----+------+-------+
  | id | a    | b     |
  +----+------+-------+
  |  1 |    1 | hello |
  |  2 |    2 | world |
  |  3 |    3 | foo   |
  |  4 |    4 | bar   |
  +----+------+-------+
  4 rows in set (0.00 sec)
  ```

- 查看 slave 同步状态:

  ```sql
  mysql> show slave status\G
  *************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                    Master_Host: mysql_master
                    Master_User: repl
                    Master_Port: 3306
                  Connect_Retry: 60
                Master_Log_File: master.000003
            Read_Master_Log_Pos: 2150
                Relay_Log_File: slave_relay-bin.000002
                  Relay_Log_Pos: 1780
          Relay_Master_Log_File: master.000003
              Slave_IO_Running: Yes
              Slave_SQL_Running: Yes
                Replicate_Do_DB: 
            Replicate_Ignore_DB: 
            Replicate_Do_Table: 
        Replicate_Ignore_Table: 
        Replicate_Wild_Do_Table: 
    Replicate_Wild_Ignore_Table: 
                    Last_Errno: 0
                    Last_Error: 
                  Skip_Counter: 0
            Exec_Master_Log_Pos: 2150
                Relay_Log_Space: 1988
                Until_Condition: None
                Until_Log_File: 
                  Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File: 
            Master_SSL_CA_Path: 
                Master_SSL_Cert: 
              Master_SSL_Cipher: 
                Master_SSL_Key: 
          Seconds_Behind_Master: 0
  Master_SSL_Verify_Server_Cert: No
                  Last_IO_Errno: 0
                  Last_IO_Error: 
                Last_SQL_Errno: 0
                Last_SQL_Error: 
    Replicate_Ignore_Server_Ids: 
              Master_Server_Id: 1
                    Master_UUID: 22da5f4a-62f5-11eb-a934-0242c0a8c865
              Master_Info_File: mysql.slave_master_info
                      SQL_Delay: 0
            SQL_Remaining_Delay: NULL
        Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
            Master_Retry_Count: 86400
                    Master_Bind: 
        Last_IO_Error_Timestamp: 
      Last_SQL_Error_Timestamp: 
                Master_SSL_Crl: 
            Master_SSL_Crlpath: 
            Retrieved_Gtid_Set: 
              Executed_Gtid_Set: 
                  Auto_Position: 0
          Replicate_Rewrite_DB: 
                  Channel_Name: 
            Master_TLS_Version: 
        Master_public_key_path: 
          Get_master_public_key: 0
              Network_Namespace: 
  1 row in set (0.00 sec)
  ```


## 复制已经保存有用户数据的服务器

还是使用之前的 docker-compose.yml 配置文件. 不过, 这里我们启动服务器后先往 master 服务器写入一些数据, 然后再让 slave 服务器复制 master 服务器.

注意要删掉之前 docker-compose 创建的容器, 例如使用 `docker-compose rm` 命令.

在复制之前, 我们需要将 master 服务器的数据导入到 slave 服务器. 这里测试两种导入数据的方法, 一种是使用 mysqldump, 另一种是复制数据库的所有文件.

步骤:

- 使用 docker-compose 启动服务器.
  
  ```sh
  docker-compose up
  ```

- 在 master 服务器创建一个用户用于复制:

  ```sql
  CREATE USER 'repl'@'%' IDENTIFIED BY 'repl1234';
  GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
  ```

- 在 master 服务器插入数据:

  ```sql
  CREATE DATABASE test;
  USE test;
  CREATE TABLE test(id INT PRIMARY KEY AUTO_INCREMENT, a INT, b CHAR(10));
  INSERT INTO test(a, b) VALUES
      (1, 'hello'),
      (2, 'world'),
      (3, 'foo'),
      (4, 'bar');
  ```

此时, master 服务器上已经写入了几条数据. 我们准备往 slave 服务器导入数据.

### 使用 mysqldump 导出数据

步骤:

- 在 slave 服务器使用 mysqldump 导出数据.

  ```sh
  mysqldump -h mysql_master -u root -p --all-databases --master-data > /db.sql
  ```

  在 db.sql 的开头可以发现这条 SQL: `CHANGE MASTER TO MASTER_LOG_FILE='master.000003', MASTER_LOG_POS=1453;`. 同时, master 服务器上的生成了一个新的 binlog 文件 master.000004.

-  设置 master 服务器信息, 以及导入 db.sql.

  进入 mysql 命令行设置 master 服务器的信息:

  ```sql
  CHANGE MASTER TO
      MASTER_HOST = 'mysql_master',
      MASTER_USER = 'repl',
      MASTER_PASSWORD = 'repl1234';
  ```

  这里可以不用手动设置 MASTER_LOG_FILE 和 MASTER_LOG_POS, 因为我们等下会执行 db.sql. 

  ```sql
  source /db.sql;
  ```

- 查看导入的数据:

  ```sql
  mysql> select * from test;
  +----+------+-------+
  | id | a    | b     |
  +----+------+-------+
  |  1 |    1 | hello |
  |  2 |    2 | world |
  |  3 |    3 | foo   |
  |  4 |    4 | bar   |
  +----+------+-------+
  4 rows in set (0.00 sec)
  ```

- 开启复制线程.

  ```sql
  START SLAVE;
  ```

- 在 master 服务器插入新的数据.

  ```sql
  INSERT INTO test(a, b) VALUES
      (5, 'apple'),
      (6, 'peach'),
      (7, 'banana'),
      (8, 'pineapple');
  ```

- 在 slave 上查看新插入的数据.

  ```sql
  mysql> select * from test;
  +----+------+-----------+
  | id | a    | b         |
  +----+------+-----------+
  |  1 |    1 | hello     |
  |  2 |    2 | world     |
  |  3 |    3 | foo       |
  |  4 |    4 | bar       |
  |  5 |    5 | apple     |
  |  6 |    6 | peach     |
  |  7 |    7 | banana    |
  |  8 |    8 | pineapple |
  +----+------+-----------+
  8 rows in set (0.00 sec)
  ```

### 复制数据库文件



## 基于 GTID 的复制