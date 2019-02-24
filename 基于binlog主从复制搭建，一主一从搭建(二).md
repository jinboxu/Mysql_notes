## 基于binlog主从复制搭建，一主一从搭建(二)

#### 事务的隔离级别

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交( read-uncommitted ) | 是   | 是         | 是   |
| 读已提交( read-committed )   | 否   | 是         | 是   |
| 可重复读( repeatable-read )  | 否   | 否         | 是   |
| 串行化( serializable )       | 否   | 否         | 否   |

幻读是指同样一笔查询在整个事务过程中多次执行后，查询 所得的结果集是不一样的。

**数据库隔离级别详解文档：https://www.jianshu.com/p/4b34c84c24e9 **



Mysql 默认隔离级别为可重复读。 

查看mysql事务隔离级别：

```mysql
mysql> show variables like '%isolation%';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.02 sec)
```



#### 不影响业务的主从搭建

- 使用可重复读隔离级别将数据dump下来

```shell
# mysqldump -h192.168.181.128 -uroot -p123456 --default-character-set=utf8 --databases testdb --single-transaction --master-data=2 > /root/test.sql
```

**single‐transaction 使用可重复读隔离级别,可以保证数据的 读一致性。innodb**

 ‐‐master‐data=1    不注释 change master 

‐‐master‐data=2    注释 change master 



- 从库初始化数据

```shell
# mysql -uroot -pflzx3qc --default-character-set=utf8 < /root/test.sql
```

```mysql
mysql> reset slave;     //清掉change master产生的信息
Query OK, 0 rows affected (0.39 sec)

mysql> change master to master_host='192.168.181.128',master_port=3306,master_user='repl',master_password='123456', \
    -> master_log_file='mysql-bin.000004',master_log_pos=1530;
Query OK, 0 rows affected, 2 warnings (0.45 sec)

mysql> start slave;
Query OK, 0 rows affected (0.09 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.181.128
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 1821
               Relay_Log_File: ubuntu-relay-bin.000002
                Relay_Log_Pos: 613
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
....(略)
```

**注意:主库处于仍在工作状态，dump下数据后(在开启主从同步之前)主库可能又写入了新的数据，所以此时需要先在主库上```show master status；```记录记下 file 和 position 字段对应的参数**

当然这种方式也会产生记录的file和position不对的状况，所以使用GTID时最好的方式

