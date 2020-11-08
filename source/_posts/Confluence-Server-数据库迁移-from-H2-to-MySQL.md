---
title: Confluence Server 数据库迁移 from H2 to MySQL
date: 2020-06-22 14:31:11
tags:
	- Confluence
	- Docker
---

## 背景
之前按照 [Docker安装Confluence](https://www.jianshu.com/p/8e81caca5f2a) 部署了 confluence server。前期为了方便直接使用了 confluence 内嵌的 H2 数据库。后续小伙伴陆陆续续开始使用起来后，为了稳定还是按照官方要求使用外部数据库，并将现有的数据进行迁移。

## 操作步骤
### 备份 confluence 
1. 使用管理员账号进入 **站点管理** -> **一般配置** -> **备份与还原**，对当前 confluence 进行备份操作

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/18/172c6bd9e19dc5ad?f=png&s=63731" width="648" height="286" />
</div>

备份完成后，会在容器 `/var/atlassian/confluence/backups/` 目录下生成相应的备份文件

2. 执行 `docker cp 8172e44b0b61:/var/atlassian/confluence/backups/xmlexport-20200619-051237-3.zip /home/confluence` 命令将站点备份文件从容器中拷贝至主机目录下

### 安装 MySQL
亲测 MySQL 8.0 不支持，最后选用 MySQL 5.7，提供两种方式进行部署

#### Host 部署 MySQL
1. 直接 apt 安装 MySQL `sudo apt-get install mysql-server`
2. 进行 MySQL 的初始化 `sudo mysql_secure_installation`
3. 进入 MySQL 终端，创建 confluence 库和用户并且授权
```
mysql> CREATE DATABASE confluence CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

mysql> CREATE USER 'confluence'@'%' IDENTIFIED BY '<password>';

mysql> GRANT ALL PRIVILEGES ON confluence.* TO 'confluence'@'%' IDENTIFIED BY '<password>';

mysql> flush privileges; 
```

4. 修改 MySQL 配置文件 

```
vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
bind-address = 0.0.0.0 #127.0.0.1
character-set-server = utf8mb4
collation-server = utf8mb4_bin
default-storage-engine = INNODB
max_allowed_packet = 256M
innodb_log_file_size = 2GB
transaction-isolation = READ-COMMITTED
```

重启 MySQL `systemctl restart mysql`

#### Docker 部署 MySQL
1. 拉取 MySQL 镜像 `docker pull mysql:5.7`
2. 创建 MySQL 容器 `docker run -p 3306:3306 --name mysql -v /data/mysql:/var/lib/mysql/ -e MYSQL_ROOT_PASSWORD=<password> -d mysql:5.7`
3. 进入容器修改配置 `docker exec -it mysql /bin/bash`
4. 创建 confluence 库和用户并且授权
```
mysql> CREATE DATABASE confluence CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

mysql> CREATE USER 'confluence'@'%' IDENTIFIED BY '<password>';

mysql> GRANT ALL PRIVILEGES ON confluence.* TO 'confluence'@'%' IDENTIFIED BY '<password>';

mysql> flush privileges; 
```
5.修改 MySQL 配置文件
```
vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
bind-address = 0.0.0.0 #127.0.0.1
character-set-server = utf8mb4
collation-server = utf8mb4_bin
default-storage-engine = INNODB
max_allowed_packet = 256M
innodb_log_file_size = 2GB
transaction-isolation = READ-COMMITTED
```
6. 重启 MySQL 容器 `docker restart mysql`

### 部署新的 confluence
1. 将之前的 confluence 容器停止 `docker stop 8172e44b0b61`
2. 创建新的 confluence 容器 `docker run -d -v /data/conflunece:/var/atlassian/confluence --name confluence -p 8090:8090 --user root:root cptactionhank/atlassian-confluence:latest `
3. 选择使用非内置数据库

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf145a3b81231?f=png&s=63589" width="603" height="328" />
</div>

4. 配置数据库连接

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf1a5de0831bc?f=png&s=71044" width="606" height="400" />
</div>

### 数据迁移
1. 在**加载内容** 页选择 **从备份还原**

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf21da53ad4ec?f=png&s=88344" width="606" height="416" />
</div>

2. 将之前站点备份文件导入 `cp /home/confluence/xmlexport-20200619-051237-3.zip /data/conflunece/restore/`

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf24a1e450e1d?f=png&s=40713" width="609" height="220" />
</div>

3. 等待数据还原

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf255a3a21f5b?f=png&s=35665" width="607" height="270" />
</div>

4. 恢复完成

<div align="center">    
    <img src="https://user-gold-cdn.xitu.io/2020/6/20/172cf2ceba712728?f=png&s=22243" width="606" height="203" />
</div>
