# MySQL学习

##### 远程连接-坑

`grant all privileges on *.* to ‘root‘@‘%‘ identified by ‘123456‘ with grant option`

只适用于mysql8.0之前的版本
之后采用这句:

```mysql
create user root@'%' identified by '123456';
 
grant all privileges on *.* to root@'%' with grant option;
FLUSH PRIVILEGES;
```

---

##### 【问题】阻塞

is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'

同一个ip在短时间内产生太多（超过mysql数据库max_connection_errors的最大值）中断的数据库连接而导致的阻塞；

方式一：

```mysql
mysql>show global variables like '%max_connect_errors%';
mysql>set global max_connect_errors=1000;
```

方式二：

`mysqladmin  -uroot -p  -h [host.ip] flush-hosts`

`[host.ip]` 例：127.0.0.1

---

##### 【问题】登录模式

msyql8.0以后登录验证插件默认为`caching_sha2_password` ，客户端5.7则是`mysql_native_password` ，二者不兼容，导致登录时：

`Authentication plugin 'caching_sha2_password' cannot be loaded:...` 

ways：

- 升级客户端到8.0以上

- ```mysql
  mysql> use mysql;
  mysql> SELECT Host, User, plugin from user;
  #根据需要将用户登录验证插件改回mysql_native_password
  mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> FLUSH PRIVILEGES;
  ```

  

