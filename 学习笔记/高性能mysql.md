## 第三章 服务器性能剖析
### 捕获MySQL的查询到日志文件中
MySQL 5.0 以上的版本可以支持把执行比较慢的SQL语句记录下来。

````
mysql> show variables like 'long%';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
1 row in set (0.01 sec)

mysql> set long_query_time =1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'slow%';
+---------------------+--------------------------------------+
| Variable_name       | Value                                |
+---------------------+--------------------------------------+
| slow_launch_time    | 2                                    |
| slow_query_log      | OFF                                  |
| slow_query_log_file | /var/lib/mysql/5d61089375c4-slow.log |
+---------------------+--------------------------------------+
3 rows in set (0.01 sec)

mysql> set global slow_query_log='ON';
Query OK, 0 rows affected (0.00 sec)
````

字段解释：

+ long_query_time 当SQL语句执行时间超过此数值时，就会被记录到日志中，建议设置为1或者更短。

+ slow_query_log 这个参数设置为ON，可以捕获执行时间超过一定数值的SQL语句。

+ slow_query_log_file 记录日志的文件名。

+ log_queries_not_using_indexes 这个参数设置为ON，可以捕获到所有未使用索引的SQL语句，尽管这个SQL语句有可能执行得挺快。

一旦slow_query_log变量被设置为ON，MySQL会立即开始记录。

也可以在 `/etc/my.cnf` 里面可以设置上面MySQL全局变量的初始值。如下所示：

````
slow_query_log
long_query_time = 1
````

解释：

+ slow_query_log 这句是开启记录慢查询功能，slow_query_log=0关闭；slow_query_log=1开启（这个1可以不写）

+ long_query_time = 1 这句是记录超过1秒的SQL执行语句

+ 那么这个日志文件存放在什么地方呢？
默认是放在mysql的data目录，并且文件名为host_name-slow.log即 主机名-slow.log,比如在笔者的开发机上就是THINK-slow.log(因为偶用的Thinkpad，呵呵)

+ 如果日志文件不想放在data目录，我们可以通过如下配置指定存放的目录及日志文件名：
slow_query_log_file=file_name
其中file_name就是你的存放日志的目录和文件名，在这里注意有的资料上可能是log-slow-queries=file_name,这个在mysql5.5版已经过时！

+ 如何记录低于1s的慢查询记录呢？
MySQL5.21版以前long_query_time 参数的单位是秒，默认值是10。这相当于说最低只能记录执行时间超过 1 秒的查询，怎么记录查询时间超过100毫秒的SQL语句记录呢？在mysql5.21+后版本支持毫秒记录

进入MySql控制台，运行如下sql语句：

`set global long_query_time=0.1`

该句是设置记录慢查询超过时间100ms的SQL，记住要重启mysql才能生效！

### 分析查询日志
从慢查询日志中生成剖析报告需要一款好工具，这里我们建议使用`pt-query-digest`。

参考：https://www.cnblogs.com/luyucheng/p/6265873.html

通过慢查询日志记录查询或者使用`pt-query-digest`分析tcpdump的结果，是可以找到的最好的方式。

### 剖析单条查询
定位到需要优化的单条查询之后，可以针对此查询钻取更多的信息。

在实际应用中，主要使用`show status`,`show profile`,检查慢查询日志的条目来分析。

#### show profile
`show profile`命令是在MySQL 5.1 以后引入的。默认是禁用的，但可以通过服务器变量在 **会话（连接）级别** 动态的修改。

`set profiling = 1;`

然后在服务器上执行的所有语句，都会测量其耗费的时间和其他一些查询执行状态变更相关的数据。

当一条查询提交给服务器时，此工具会记录剖析信息到一张临时表，并且给查询赋予一个从 1 开始的整数标识符。

剖析报告给出了查询执行的每个步骤及其花费的时间。

也可以不适用`show profile`命令，而直接查询`INFORMATION_SCHEMA`中的`PROFILING`表。

````
select 
````
