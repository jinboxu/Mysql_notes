## 基于binlog主从复制搭建，一主一从搭建(一)

Mysql主从复制的实现原理图大致如下 

![mysql主从](D:\github_projects\mysql笔记\pics\mysql主从.jpg)





- 主从两台主机都安装了mysql，并正常运行状态
- 设置字符集和master开启binlog:

查看字符集:

```mysql>show variables like '%char%';```

查看binlog:

```mysql>show variables like '%log_bin%';```

主库需要开启 binlog,从库不是必须的。 



**修改主配置文件:**

[mysqld]
character_set_server=utf8
server-id = 101
log-bin = mysql-bin

**修改从配置文件：**

[mysqld]
character_set_server=utf8
server-id = 102



重启主从mysql服务后，达到设置要求。



- 在主库添加一个用户 repl 并指定 replication 权限 

```mysql>grant replication slave on *.* TO 'repl'@'%' identified by '123456';```



- 在主数据库里面运行 show master status;记下 file 和 position 字段对应的参数。 

``````mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     660 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
``````



- 在从库设置它的master

```mysql
mysql> change master to master_host='192.168.181.128',master_port=3306,master_user='repl',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=660;
```

这里的 master_log_file 和 master_log_pos 对应刚才 show master status 记下的参数。 



- 在从库开启从数据库复制功能，并通过```show slave status\G```来查看

```mysql
mysql> start slave;
Query OK, 0 rows affected (0.06 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.181.128
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 155
               Relay_Log_File: ubuntu-relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes     ####1
            Slave_SQL_Running: Yes     ####2
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
.......(等，未显示完)
mysql> show processlist;       //查看mysql当前的进程
```

至此一个简单的主从复制架构我们就完成了 



#### 在搭建主从中遇到的问题

> 主从搭建完成后，在从上执行```show slave status\G```,发现Slave_IO_Running: No，通过查看日志
>
> 2019-02-19T03:02:08.682818Z 12 [ERROR][MY-013117] [Repl] Slave I/O for channel '': Fatal error: The slave I/O thread stops because master and slave have equal MySQL server UUIDs; these UUIDs must be different for replication to work. Error_code: MY-013117
>
> 即:从服务器是通过服务器虚拟机复制出来的，这样主从服务器的mysql的uuid是一样的，修改从服务器mysql的uuid并重启mysql服务，ok





