# MySQL Replication Playground


## 设置全新的主从复制

启动两个全新的 MySQL 服务器, 一开始时它们都没有写入任何数据.

使用 docker-compose 来创建和管理服务器, 配置文件在这里[docker-compose.yml](./docker-compose.yml).

- 使用 docker-compose 启动服务器.
  
  ```sh
  docker-compose up
  ```

- 在 master 服务器创建一个用户用于复制:

  ```sql
  CREATE USER 'repl'@'*' IDENTIFIED BY 'repl1234';
  GRANT REPLICATION SLAVE ON *.* TO 'repl'@'*';
  ```

- 查看 master 服务器的 binlog 信息:

  ```sql
  mysql> show master status;
  +---------------+----------+--------------+------------------+-------------------+
  | File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
  +---------------+----------+--------------+------------------+-------------------+
  | master.000003 |      697 |              |                  |                   |
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
      MASTER_LOG_POS = 697;

  START SLAVE;
  ```

  注意上面 SQL 中的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 是前面在 master 服务器上执行 `show master status` 的结果.

- 在 master 服务器插入数据:

  ```sql
  CREATE DATABASE test;
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
                    Master_User: root
                    Master_Port: 3306
                  Connect_Retry: 60
                Master_Log_File: master.000003
            Read_Master_Log_Pos: 1453
                Relay_Log_File: mysql_slave-relay-bin.000002
                  Relay_Log_Pos: 1075
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
            Exec_Master_Log_Pos: 1453
                Relay_Log_Space: 1289
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
                    Master_UUID: 50a74335-62e4-11eb-850d-0242c0a8c865
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


