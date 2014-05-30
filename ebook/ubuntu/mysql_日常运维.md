# mysql 日常运维


## mysql安装

```

```


##innobackupex备份

###以流的方式备份

接收端

```
mkdir -p /data/backup
cd /data/backup
nohup nc -d -k -l 5555 | nohup xbstream -x - &
```

发送端

```
nohup innobackupex   --safe-slave-backup --no-lock --slave-info --defaults-file=/etc/mysql/my.cnf --user=root --password='xxx'   --stream=xbstream --databases="test mysql"  ./  2>/data/tmp/a.log| nohup cat - | nohup nc 10.10.x.x 5555 &
```
其中， 

--safe-slave-backup --no-lock --slave-info 备份的文件可以做从服务器
--databases 指定只备份若干个库

###恢复数据
恢复数据

```
mkdir -p /data/mysqldb
mysql_install_db 
mv  /data/mysqldb /data/mysqldb_org

mkdir -p /data/mysqldb
innobackupex --databases="test mysql"   --apply-log   /data/backup/xxxxx-xx-xx
innobackupex --databases="test mysql"   --copy-back /data/backup/xxxxx-xx-xx


cp -r /data/mysqldb_org/performance_schema/ /data/mysqldb

chown -R mysql:mysql /data/mysqldb

 启动mysql
/etc/init.d/mysql start
```

###设为从服务器
查看同步的二进制文件名和位置

```
cat /data/backup/xxxxx-xx-xx/xtrabackup_binlog_pos_innodb 
```

然后设置

```
stop slave;
change master to master_user='rep', master_password='xxxxx', master_host='1xx.xx.xx.xx',  master_log_file='mysql-bin.00xxx', master_log_pos=xxx;
start slave;
show slave status\G
```

#### 如果只同步部分库，要把不需要同步的库的表结构在新数据库创建一下

在原数据库上

```
/usr/local/mysql/bin/mysqldump -uroot -pxxxx  --skip-opt --single-transaction --add-drop-table --create-options --quick --extended-insert --set-charset --disable-keys --no-data  test001 >test001.sql
```

在备份数据库上， 取到生成sql文件， 执行下面语句

```
mysql> create database caihui;
mysql> use caihui
mysql> source caihui.sql;

```

然后重启下slave

```
mysql>stop slave;
mysql>start slave;
```

--------------

## Slave 跳过堵住的 SQL

```
mysql> STOP SLAVE;
mysql> SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; 
mysql> START SLAVE;
```
### Change Data Dir

```
stop mysql
cp -R -p /var/lib/mysql /data/mysql
vi /etc/mysql/my.cnf # change data_dir
vi  /etc/apparmor.d/usr.sbin.mysqld # change file permission vi /etc/apparmor.d/abstractions/mysql # change file permission
restart apparmor
rm -rf /var/run/mysqld
start mysql
```

----
## 常用sql

```
CREATE DATABASE test02;
USE test02;
CREATE TABLE example (node_id INT PRIMARY KEY, node_name VARCHAR(30));
INSERT INTO test02.example VALUES (1, 'percona1');
SELECT * FROM test02.example;
```

```
use mysql
Delete FROM user Where User='debian-sys-maint' and Host='localhost';
 grant   all   privileges   on   *.*   to  'debian-sys-maint'@'localhost' IDENTIFIED BY '123456';
flush privileges; 
```

```
use snowball
select * from status order by  created_at desc limit 1;
```

------
## /etc/mysql/my.cnf

在slave上忽略一些库的同步

```
replicate-ignore-db = information_schema, performance_schema, test
```

mysql启动时默认不开启slave同步线程

```
skip-slave-start
```