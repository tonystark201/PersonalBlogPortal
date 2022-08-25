## 📖 自动初始化MySQL实例

有时候我们在使用docker安装MySQL数据的时候，需要做一些初始化工作，比如创建Mysql用户，初始化数据库表结构。

本文主要通过两种方式演示如何在 Docker 中安装 MySQL 时自动执行初始化数据库脚本。

### 🧱 W1: Custom shell script

#### Dockerfile

通过 MySQL 基础镜像自定义我们需要的容器镜像

```dockerfile
FROM mysql:5.7
MAINTAINER TSZ201 <xxxxxx@xxxxxx.com>
ENV MYSQL_ALLOW_EMPTY_PASSWORD yes
RUN mkdir -p /var/lib/mysql /home/mysql && \
    chown 777  /var/lib/mysql /home/mysql
COPY setup.sh /mysql/setup.sh
COPY schema.sql /mysql/schema.sql
COPY privileges.sql /mysql/privileges.sql
CMD ["sh", "/mysql/setup.sh"]
```

+ `setup.sh`是我们要执行的shell脚本
+ `schema.sql`和`privileges.sql`是数据库初始化SQL脚本文件

#### Setup Shell

```shell
#!/bin/bash
echo `service mysql status`
echo '1.start mysql....'
service mysql start
sleep 3s
echo `service mysql status`
echo '2.begin init table....'
mysql < /mysql/schema.sql
echo '3.end init table....'
sleep 1s
echo `service mysql status`
echo '4.begin modify user and password....'
mysql < /mysql/privileges.sql
echo '5.end modify user and password....'
sleep 1s
echo `service mysql status`
echo 'mysql contianer done'
tail -f /dev/null
```

#### SQL Schema

+ 创建表

```sql
-- create database
create database `canal_demo` default character set utf8 collate utf8_general_ci;
use canal_demo;

-- create table
DROP TABLE IF EXISTS `students` ;
CREATE TABLE IF NOT EXISTS `students` (
  `id` CHAR(22) NOT NULL COMMENT 'user id',
  `name` VARCHAR(64) NOT NULL COMMENT 'user name',
  `phone` CHAR(11) NOT NULL COMMENT 'user phone',
  `teacher_id` CHAR(22) NOT NULL COMMENT 'teacher id',
  `class_id` CHAR(22) NOT NULL COMMENT 'class id',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT 'created time',
  PRIMARY KEY (`id`))
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8
COMMENT = 'student table';
```

+ 修改权限

```sql
use mysql;
set password for root@localhost = password('123456');
grant all on *.* to root@'%' identified by '123456' with grant option;
create user canal identified by 'canal';    
grant SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';  
flush privileges;
```

__当我们执行“Dockerfile”编译镜像并启动容器时，会自动执行脚本文件，完成数据库初始化。__

### 🧱 W2: Based on the official mirror

官方镜像参考：[Here](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/Dockerfile)

以下内容摘选自官方文档：`docker-entrypoint.sh`

```shell
echo  
for f in /docker-entrypoint-initdb.d/*; do
   case "$f" in    
    *.sh)     echo "$0: running $f"; . "$f" ;; 
    *.sql)    echo "$0: running $f"; "${mysql[@]}" < "$f"; echo ;;
    *.sql.gz) echo "$0: running $f"; gunzip -c "$f" | "${mysql[@]}"; echo ;;    
    *)        echo "$0: ignoring $f" ;;   
   esac   
   echo  
done
```



如果我们仔细研究官方的MySQL镜像文件，我们可以发现它实际上支持数据库初始化脚本文件的执行。因此，我们只需要在本地编辑 SQL 脚本，并将卷挂载到 docker 的这个目录（`docker-entrypoint-initdb.d`）即可。

这里启动了两个MySQL服务，第一个服务用第一种方式初始化，第二个服务用第二种方式初始化。 我们将 MySQL 配置映射到对应目录，并将 MySQL 服务的数据映射到本地进行持久化。

DockerCompose 文档如下所示：

```yaml
version: '3'

services:
  mysql1:
    restart: always
    build: mysql
    container_name: mysql1
    ports:
      - 3310:3306
    volumes:
      - D:\mysql\data1:/var/lib/mysql
      - D:\mysql\conf\my.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
    networks:
      mysql:
        ipv4_address: 172.11.0.2
        
  mysql2:
    image: mysql:5.7
    restart: always
    container_name: mysql2
    ports:
      - 3311:3306
    volumes:
      - D:\mysql\conf\my.cnf:/etc/my.cnf
      - D:\mysql\docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
      - D:\mysql\data2:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
    networks:
      mysql:
        ipv4_address: 172.11.0.3

networks:
   mysql:
      ipam:
         config:
         - subnet: 172.11.0.0/16
```

MySQL配置文件如下所示：

```
[mysqld]
port=3306
server_id = 1
log-bin = mysql-bin
binlog-format=ROW 
binlog-do-db = students          
log-slave-updates                
sync_binlog = 1                  
auto_increment_offset = 1
auto_increment_increment = 1
expire_logs_days = 7
log_bin_trust_function_creators = 1
max_connections=200
max_connect_errors=10
character-set-client-handshake = FALSE
character-set-server = utf8
external-locking = FALSE
default-storage-engine=INNODB 
default_authentication_plugin=mysql_native_password
bind-address = 0.0.0.0
```

### 🧱 Summary

MySQL是使用比较广泛的关系型数据库，也有很多替代产品。本文主要介绍在docker中部署自动数据库初始化的两种方式，个人推荐采用第二种方式更为简便。
