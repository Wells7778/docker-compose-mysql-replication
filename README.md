# 用Docker Compose設定MySQL GROUP replication
---
## docker-compose.yml內使用版本為3.5，請留意Docker Engine版本須為	17.12.0+
### 步驟一：先開啟兩個mysql容器
> 因為本例使用的機器是MacBook Pro Silicon，所以docker-compose.yml內有設定platform為linux/amd64，這樣才抓得到正確的mysql image，如果你不是使用M1晶片的機器，這個platform請自行刪除
```
docker-compose up -d
```
等待兩個container都開啟
```
docker-compose ps
```
容器狀態類似如下：
```
CONTAINER ID   IMAGE       COMMAND                  CREATED             STATUS                       PORTS                                                  NAMES
container_id   mysql:5.7   "docker-entrypoint.s…"   About an hour ago   Up About an hour (healthy)   33060/tcp, 0.0.0.0:3308->3306/tcp, :::3308->3306/tcp   replica
container_id   mysql:5.7   "docker-entrypoint.s…"   About an hour ago   Up About an hour (healthy)   33060/tcp, 0.0.0.0:3307->3306/tcp, :::3307->3306/tcp   main
```

### 步驟二：在main(master)設定replication帳號和權限
先利用docker-compose exec連進main這個容器內的mysql
```
docker-compose exec main mysql -u root -p
```
接著設定帳號與權限
```
mysql> CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
mysql> FLUSH PRIVILEGES;
```
然後透過指令記下main執行狀態
```
mysql> SHOW MASTER STATUS\G;
```
顯示畫面類似：
```
*************************** 1. row ***************************
             File: mysql-bin.000006
         Position: 3994
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
```
請記下**File**和**position**，之後需要設定到replica裡

### 步驟三：在replica(slave)設定replication group
一樣透過docker-compose exec連進replica這個容器內的mysql
```
docker-compose exec replica mysql -u root -p
```
先停掉SLAVE
```
mysql> STOP SLAVE;
```
接著把master資訊設定進去
> 剛剛記下來的File填到MASTER_LOG_FILE裡，position填到MASTER_LOG_POS裡
```
mysql> CHANGE MASTER TO MASTER_HOST='172.24.0.2', MASTER_PORT=3306, MASTER_USER='replication', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=504;
```
開啟SLAVE
```
mysql> START SLAVE;
```
最後檢查同步狀態
```
mysql> SHOW SLAVE STATUS\G;
```
結果會類似如下：
```
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.24.0.2
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000006
          Read_Master_Log_Pos: 3994
               Relay_Log_File: replica-relay-bin.000002
                Relay_Log_Pos: 4160
        Relay_Master_Log_File: mysql-bin.000006
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
          Exec_Master_Log_Pos: 3994
              Relay_Log_Space: 4369
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
                  Master_UUID: 18dcd1b6-169e-11ec-891c-93afb068f593
             Master_Info_File: /var/lib/mysql/master.info
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
```

只要確認這兩個值是Yes就是已經正常啟動了，可以試著在main建立資料就會自動同步到replica
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```