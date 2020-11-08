---
title: MySQL Binlog Replication
date: 2020-01-05 21:32:41
tags: 
	- MySQL 
	- Binlog
---

MySQL 开启 binlog 配置主从同步步骤整理。


## 示例服务器准备

Azure VM 上创建两台 centos 7.5 服务器分别作为 MySQL master 和 My SQL slave：

* master: 13.75.108.150
* slave: 13.75.109.41

## 操作步骤
### 安装 MySQL 
#### 添加 MySQL 5.7仓库

```
sudo rpm -ivh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

#### 安装 MySQL 5.7

```
sudo yum -y install mysql-community-server
```

#### MySQL 服务配置

##### 启动 MySQL 服务

```
sudo systemctl start mysqld
```

##### 设置开启自启

```
sudo systemctl enable mysqld
```

#### 安全设置

##### 获取 MySQL 临时密码

```
cat /var/log/mysqld.log | grep -i 'temporary password'
```

##### 配置安全设置

```
mysql_secure_installation
```

### 配置 binlog 主从同步

#### master 服务器配置

##### 创建同步用户

```
mysql> CREATE USER 'slave'@'%' IDENTIFIED BY '{YOUR PASSWORD}';

mysql> GRANT ALL PRIVILEGES ON *.* TO slave@"%" IDENTIFIED BY "{YOUR PASSWORD}";
```

##### 开启 binlog

修改 `/etc/my.cnf` 文件，添加内容如下：

```
log-bin=mysql-bin
server-id=10
```

重启 MySQL 服务：

```
systemctl restart mysqld.service
```

##### 获取初始同步位置

```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

#### slave 服务器配置

##### 修改 MySQL 配置

修改 `/etc/my.cnf` 文件，添加内容如下：

```
server-id=12
replicate-do-db=test
skip-slave-start=true
```

重启 MySQL 服务：

```
systemctl restart mysqld.service
```

##### 配置主从同步

停止 slave 服务进程：

```
mysql> stop slave;
```

配置同步上游信息：

```
mysql> change master to 
master_host='13.75.108.150',
master_user='slave',
master_password='{YOUR PASSWORD}',
master_log_file='mysql-bin.000001', 
master_log_pos=313;
```

开启 slave 服务进程：

```
mysql> start slave;
```

查看 slave 同步状态：

```
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 13.75.108.150
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 1064
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 1071
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: test
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1064
              Relay_Log_Space: 1278
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
             Master_Server_Id: 10
                  Master_UUID: 8c019b28-306a-11ea-80a3-000d3a8507a5
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