### 在Docker中配置MySQL的主从同步

Docker能非常方便的运行多个MySQL容器，我们可以使用docker来搭建一个MySQL主从同步。需要有docker环境。

<br desc/>

#### 下载并运行MySQL

```shell
docker pull mysql:5.6

#启动主MySQL
docker run --name mysql-master -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:5.6
#启动从MySQL
docker run --name mysql-slave -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:5.6
```

注意两个容器映射的宿主机端口不能相同，这里使用 3306表示master，3307表示slave。

通过`docker ps -a`命令查看是否正常启动了容器。

<br desc/>

#### 主（master）配置

```shell
#进入主MySQL容器内部
docker exec -it mysql-master bash
```

##### 安装vim

```shell
apt-get update
apt-get install vim -y
```

##### 更改配置文件

编辑`vi /etc/mysql/my.cnf`文件，添加如下内容：

```shell
[mysqld]
## 设置server-id，同一局域网要唯一
server-id=1  
## 开启二进制日志
log-bin=mysql-bin
```

保存退出之后通过`service mysql restart`命令重启MySQL，重置之后会断开docker容器，需要重新进入容器。

##### 创建数据同步用户

用于主从数据库之间同步数据。

```shell
#进入MySQL
mysql -u root -h localhost -p
#输入密码进入
```

创建用户，授予用户REPLICATION SLAVE权限和REPLICATION CLIENT权限

```mysql
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```

##### 获取链接File和Position信息

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      433 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```



#### 从（slave）配置

编辑`vi /etc/mysql/my.cnf`文件，添加如下内容：

```shell
[mysqld]
## 设置server-id，同一局域网要唯一
server-id=1  
## 开启二进制日志
log-bin=mysql-bin
```

保存退出之后通过`service mysql restart`命令重启MySQL，同样重新进入容器。

##### 执行同步SQL语句

```mysql
mysql> change master to
    -> master_host='172.17.0.3',
    -> master_user='slave',
    -> master_password='123456',
    -> master_port=3306,
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=433,
    -> master_connect_retry=30;
Query OK, 0 rows affected, 2 warnings (0.02 sec)
```

> 参数说明：
>
> **master_host**：master容器的ip，通过`docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql5.6`可获取
>
> **master_user**：用于数据同步的用户，master创建
>
> **master_password**：用于同步的用户的密码，master创建时指定
>
> **master_port**：master容器内的端口号
>
> **master_log_file**：指 slave 从master的哪个日志文件开始复制数据，上文中的File字段
>
> **master_log_pos**：指slave从哪个 Position 开始读，上文中的 Position 字段
>
> **master_connect_retry**：重连时间间隔，单位是秒，默认是60秒

##### 开启slave同步进程

```mysql
mysql> start slave;
```



##### 查看连接状态

```mysql
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.3
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 433
               Relay_Log_File: edu-mysql-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
......
#省略下边内容
```

确保`Slave_IO_Running`、`Slave_SQL_Running`都显示为Yes



#### 测试

到此为止，主从配置已完成，可以自行插入数据来测试看看结果。