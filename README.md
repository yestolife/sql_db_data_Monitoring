# 对SQL数据库中数据的监控

在做数据可视化展示过程中，常常需要展示数据库中的一些数据。搜索了一圈中文世界，很多是对数据库性能的监控展示，缺少对数据本身监控的实例说明。我们就来实现一下对sql类数据库中数据本身的监控。

## 1. 实验环境：

1. [ubuntu 18.04](http://releases.ubuntu.com/18.04/)操作系统
2. [Mysql 8.0.17](https://www.runoob.com/mysql/mysql-install.html)，在mysql数据库中展示数据监控，其实sql类的数据库均可实现
3. [sql_exporter-0.5.linux-386 ](https://github.com/free/sql_exporter/releases)，该工具定时从sql数据库取数，并将数据暴露给Prometheus
4. [Prometheus 2.11.1](https://prometheus.io/docs/prometheus/latest/installation/)，开源的数据采集工具，有基本的数据展示功能

## 2.mysql数据库配置

`create database test_db`创建一个名为`test_db`的新库;

`use test_db`选择库`test_db`;

`create table test_tab (test_col int(1))`创建表`test_tab`;

`insert into test_tab (test_col) values (1)`写入数据；

数据配置完毕，查询一下：

```
mysql> select * from test_tab;
+----------+
| test_col |
+----------+
|        1 |
+----------+
```

由于mysql数据库在8.0以上版本修改了加密方式，使用Go-SQL-Driver读取数据库是会报错：

```
this authentication plugin is not supported.
```

所以还需要进行一项[配置](https://blog.csdn.net/Charliewolf/article/details/82556583)，首先用`root`用户进入mysql，修改`root`用户的密码和加密方式：这里修改密码为`abc345`:

```
mysql> alter user root@localhost identified with mysql_native_password by 'abc345'
```

## 3. sql_exporter配置

sql_exporter相关两个文件需要配置sql_exporter.yml与mysql_standard.collector.yml。[sql_exporter.yml](https://github.com/yestolife/sql_db_data_Monitoring/blob/master/sql_exporter.yml)中保存了数据库链接的信息，特别注意`data_source_name: 'mysql://root:abc345@tcp(127.0.0.1:3306)/test_db'`需要与本机mysql数据库配置保持一致。[mysql_standard.collector.yml](https://github.com/yestolife/sql_db_data_Monitoring/blob/master/mysql_standard.collector.yml)中保存了读取数据库中数据的查询方式。

运行sql_exporter后可以看到数据暴露于端口9399。

```
$ ./sql_exporter
I0817 16:56:52.568408    1035 main.go:52] Starting SQL exporter (version=0.5, branch=master, revision=fc5ed07ee38c5b90bab285392c43edfe32d271c5) (go=go1.11.3, user=root@f24ba5099571, date=20190114-09:24:17)
I0817 16:56:52.569146    1035 config.go:18] Loading configuration from sql_exporter.yml
I0817 16:56:52.569564    1035 config.go:131] Loaded collector "mysql_standard" from mysql_standard.collector.yml
I0817 16:56:52.569647    1035 main.go:67] Listening on :9399
```

## 4. Prometheus配置

Prometheus的配置比较简单，[prometheus.yml](https://github.com/yestolife/sql_db_data_Monitoring/blob/master/prometheus.yml)文件中的一些参数,注意侦听的端口与sql_exporter匹配：

```
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'mysql'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9399']
```

运行Prometheus后，打开浏览器，输入127.0.0.1;9090可查看数据监控情况



![监控链接成功](prom1.png)

![数据读取成功](prom2.png)

从上图可以看到，读出了数据库中test_col字段的数据。在数据库中修改一下，看看数据变化能否监控。

```
mysql> update test_tab set test_col=2;
Query OK, 1 row affected (0.08 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from test_tab;
+----------+
| test_col |
+----------+
|        2 |
+----------+
1 row in set (0.00 sec)
```

在打开Prometheus的监控页面

![数据变化也能监控到](prom3.png)



## 总结

我们利用sql_exporter+Prometheus实现了对mysql数据库中数据的监控，并将数据可视化展现，数据变化一目了然。除了mysql数据库，还可对sql类的其他数据库进行监控。下一步，将研究对NoSQL类数据库数据的监控。
