## 基于GTID技术搭建主从复制

**GTID (Global Transaction ID)，也就是全局事务 ID, 其保证为 每一个在 master 主上提交的事务在复制集群中可以生成一 个唯一的 ID。基于 GTID 的复制是从 MySQL5.6 开始支持的一种新的复制方 式，此方式与传统基于 binlog 日志的方式存在很大的差异， 在原来的基于日志的复制中，slave 从服务器连接到 master 主服务器并告诉主服务器要从哪个二进制日志的偏移量开 始执行增量同步，这时我们如果指定的日志偏移量不对，这 与可能造成主从数据的不一致，而基于 GTID 的复制会避免。** 

**在基于 GTID 的复制中，首先从服务器会告诉主服务器已经 在从服务器执行完了哪些事务的 GTID 值，然后主库会有把 所有没有在从库上执行的事务，发送到从库上进行执行，并 且使用 GTID 的复制可以保证同一个事务只在指定的从库上 执行一次，这样可以避免由于偏移量的问题造成数据不一致。**

 

**GTID=source_id:transaction_id**

1. source_id 就是执行事务的主库的 server‐uuid 值 ,server‐uuid 值是在 mysql 服务首次启动生成的，保存在数据库的数据目录中，在数据目录中有一个 auto.conf 文件 

```shell
root@ubuntu:~# cd /var/lib/mysql/
root@ubuntu:/var/lib/mysql# cat auto.cnf 
[auto]
server-uuid=f1ab793f-6797-11e8-ad81-000c292daddb
```



```mysql
mysql> show variables like '%uuid%';
+---------------+--------------------------------------+
| Variable_name | Value                                |
+---------------+--------------------------------------+
| server_uuid   | f1ab793f-6797-11e8-ad81-000c292daddb |
+---------------+--------------------------------------+
1 row in set (0.18 sec)
```



2. 事务 ID 则是从 1 开始自增的序列，表示这个事务是在主库上 执行的第几个事务 



#### 基于 GTID 技术不影响业务配置主从 

- 主库修改配置文件my.cnf: (Binlog日志还是需要开启的)

**已有的业务主库肯定已经运行了这些配置，只需设置新加入的从库**

```shell
[mysqld]
character_set_server=utf8
server-id = 101
log-bin = mysql-bin
gtid-mode = ON
enforce-gtid-consistency = ON
```



```mysql
mysql> show variables like '%gtid%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed                    |           |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
9 rows in set (0.00 sec)
```



- 将主库的的一个或多个库dump下来, 并在从库初始化数据

```shell
root@ubuntu:~# mysqldump -h192.168.181.128 -uroot -p123456 --default-character-set=utf8 --databases testdb --single-transaction --master-data=2 \
> --triggers --routines --events > /tmp/test2.sql
```



初始化数据：

```mysql
mysql> reset slave;
Query OK, 0 rows affected (0.38 sec)

mysql> reset master;    //清空 Mysql 中的 gtid_executed!!
Query OK, 0 rows affected (0.03 sec)
```



```shell
root@ubuntu:~# mysql -uroot -pflzx3qc --default-character-set=utf8 < /tmp/test2.sql 
mysql: [Warning] Using a password on the command line interface can be insecure.
```



> **为什么要在从库上执行```reset master```？**
>
> 比如我现在的环境：我之前从库已经在gtid模式下运行了一段时间，现在我停掉了slave，
>
> 并将testdb库复制了下来，如此在我初始化从库的时候就报错了
>
> ```shell
> root@ubuntu:~# mysql -uroot -pflzx3qc --default-character-set=utf8 < /tmp/test2.sql 
> mysql: [Warning] Using a password on the command line interface can be insecure.
> ERROR 3546 (HY000) at line 24: @@GLOBAL.GTID_PURGED cannot be changed: the added gtid set must not overlap with @@GLOBAL.GTID_EXECUTED
> ```
>
> 提示我该文件的gtid和我当前的环境重叠了！
>
> ```mysql
> mysql> show variables like '%gtid%';
> +----------------------------------+------------------------------------------+
> | Variable_name                    | Value                                    |
> +----------------------------------+------------------------------------------+
> | binlog_gtid_simple_recovery      | ON                                       |
> | enforce_gtid_consistency         | ON                                       |
> | gtid_executed                    | f1ab793f-6797-11e8-ad81-000c292daddb:1-2 |
> | gtid_executed_compression_period | 1000                                     |
> | gtid_mode                        | ON                                       |
> | gtid_next                        | AUTOMATIC                                |
> | gtid_owned                       |                                          |
> | gtid_purged                      | f1ab793f-6797-11e8-ad81-000c292daddb:1-2 |
> | session_track_gtids              | OFF                                      |
> +----------------------------------+------------------------------------------+
> 9 rows in set (0.02 sec)
> 
> mysql> reset master;
> Query OK, 0 rows affected (0.32 sec)
> 
> mysql> show variables like '%gtid_executed%';
> +----------------------------------+-------+
> | Variable_name                    | Value |
> +----------------------------------+-------+
> | gtid_executed                    |       |
> | gtid_executed_compression_period | 1000  |
> +----------------------------------+-------+
> 2 rows in set (0.01 sec)
> ```



- ```start slave;```

```mysql
mysql> change master to master_host='192.168.181.128',master_port=3306,master_user='repl',master_password='123456',master_auto_position=1;
Query OK, 0 rows affected, 2 warnings (0.39 sec)

mysql> start slave;
Query OK, 0 rows affected (0.10 sec)

mysql> show slave status\G
...
```



