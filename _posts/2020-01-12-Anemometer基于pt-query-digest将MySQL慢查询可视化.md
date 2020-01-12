---
layout:     post
title:      Anemometer基于pt-query-digest将MySQL慢查询可视化
subtitle:   
date:       2020-01-12
author:     BY
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
---

 ###  1 Mysql 慢查询日志

MySQL慢日志想必大家或多或少都有听说，主要是用来记录MySQL中长时间执行（超过long_query_time 单位秒），同时examine的行数超过min_examined_row_limit ,影响MySQL性能的SQL语句，以便DBA进行优化。

在MySQL中，如果一个SQL需要长时间等待获取一把锁，那么这段获取锁的时间并不算执行时间，当SQL执行完成，释放相应的锁，才会记录到慢日志中，所以MySQL的慢日志中记录的顺序和实际的执行顺序可能不大一样。

 在默认情况下，MySQL的慢日志记录是关闭的,我们可以通过将设置slow_query_log=1来打开MySQL的慢查询日志，通过slow_query_log_file=file_name来设置慢查询的文件名，如果文件名没有设置，他的默认名字为 *host_name-slow.log*。同时，我们也可以设置 log-output={FILE|TABLE}来指定慢日志是写到文件还是数据库里面（如果设置log-output=NONE，将不进行慢日志记录，即使slow_query_log=1）。

MySQL的管理维护命令的慢SQL并不会被记录到MySQL慢日志中。常见的管理维护命令包括ALTER TABLE,ANALYZE TABLE, CHECK TABLE, CREATE INDEX, DROP INDEX, OPTIMIZE TABLE, 和REPAIR TABLE。如果希望MySQL的慢日志记录这类长时间执行的命令，可以设置log_slow_admin_statements 为1。

通过设置log_queries_not_using_indexes=1，MySQL的慢日志也能记录那些不使用索引的SQL（并不需要超过long_query_time，两者条件满足一个即可）。但打开该选项的时候，如果你的数据库中存在大量没有使用索引的SQL，那么MySQL慢日志的记录量将非常大，所以通常还需要设置参数log_throttle_queries_not_using_indexes 。默认情况下，该参数为0，表示不限制，当设置改参数为大于0的值的时候，表示MySQL在一分钟内记录的不使用索引的SQL的数量，来避免慢日志记录过多的该类SQL.

在MySQL 5.7.2 之后，如果设置了慢日志是写到文件里，需要设置log_timestamps 来控制写入到慢日志文件里面的时区（该参数同时影响general日志和err日志）。如果设置慢日志是写入到数据库中，该参数将不产生作用。

所以，总结下哪些SQL能被MySQL慢日志记录：

- 不会记录MySQL中的管理维护命令，除非明确设置log_slow_admin_statements=1;
- SQL执行时间必须超过long_query_time，（不包括锁等待时间）
- 参数log_queries_not_using_indexes设置为1，且SQL没有用到索引，同时没有超过log_throttle_queries_not_using_indexes 参数的设定。
- 查询examine的行数必须超过min_examined_row_limit

注：如果表没有记录或者只有1条记录，优化器觉得走索引并不能提升效率，即使设置了log_queries_not_using_indexes=1，那么也不会记录到慢日志中。

注：如果SQL使用了QC，那也不会记录到慢日志中。

注：修改密码之类的维护操作，密码部分将会被星号代替，避免明文显示。

### 2 Anemometer 简介：

项目地址：https://github.com/box/Anemometer

演示地址：http://lab.fordba.com/anemometer/

Anemometer 是一个图形化显示从MySQL慢日志的工具。结合pt-query-digest，Anemometer可以很轻松的帮你去分析慢查询日志，让你很容易就能找到哪些SQL需要优化。

如果你想要使用Anemometer这个工具，那么你需要准备以下环境：

1. 一个用来存储分析数据的MySQL数据库
2. [pt-query-digest](http://www.percona.com/doc/percona-toolkit/pt-query-digest.html).                       (doc: [Percona Toolkit](http://www.percona.com/doc/percona-toolkit) )
3. MySQL数据库的慢查询日志 (doc: [The Slow Query Log](http://dev.mysql.com/doc/refman/5.5/en/slow-query-log.html) )
4.  PHP版本为 5.5+  apache或者nginx等web服务器均可。

**1 安装anemometer**

```javascript
cd /data/www/web3
git clone https://github.com/box/Anemometer.gitanemometer && cd anemometer
```

2 创建表和用户名

```javascript
 mysql -uroot -proot <install.sql
 mysql -uroot -proot -e"grant ALL ON slow_query_log.* to 'anemometer'@'localhost' IDENTIFIED BY '123456';"
 mysql -uroot -proot -e"grant SELECT ON *.* to 'anemometer'@'localhost' IDENTIFIED BY '123456';"
 mysql -uroot -proot -e"flush privileges;"
```

 我们可以看下表结构如下

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/ypl2l7gcd6.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/dsn095sc3y.png?imageView2/2/w/1620)

**3 分析mysql慢日志**

 pt版本高于2.2的执行下面语句，将慢查询日志放入名为slow_query_log数据库中

```javascript
 pt-query-digest --user=anemometer -h 127.0.0.1 --password=123456 \
--review h=localhost,D=slow_query_log,t=global_query_review\
--history h=localhost,D=slow_query_log,t=global_query_review_history\
--no-report --limit=0% --filter=" \$event->{Bytes} = length(\$event->{arg}) and\$event->{hostname}=\"$HOSTNAME\"" /usr/local/mariadb/var/localhost-slow.log
```

这时候，数据库的slow_query_log 库，里面的global_query_review_history和global_query_review这2张表已经已经有一些数据了。

**4 修改anemometer配置文件及配置展示日志用的虚拟主机**

```javascript
 cd /data/www/web3/anemometer/conf
 cp sample.config.inc.php  config.inc.php
 vim config.inc.php  主要修改的地方如下2个：
```

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/wj62snu4po.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/y4par7d3qq.png?imageView2/2/w/1620)

**5 配置nginx**

\# vim /usr/local/nginx/conf/vhost/anemometer.conf  内容如下：

```javascript
server {
       listen   80;
       server_name  192.168.0.88;
       access_log  /home/wwwlogs/anemometer.log  access;
       index index.php index.html;
       root  /data/web3/anemometer;
       include enable-php.conf;
}
/etc/init.d/nginx reload      重载nginx配置文件
```

在浏览器访问http://192.168.0.88/ 即可如下图所示(这几张图片是从别人博客摘录的，他这个截图做的特别详细)

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/7kkk4h4pjb.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/pbrmykfnrg.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/m12x2d67lk.png?imageView2/2/w/1620)

**5 自动滚动日志**

 vi /etc/logrotate.d/mysql

```javascript
postrotate
pt-query-digest --user=anemometer --password=123456 \
--review D=slow_query_log,t=global_query_review \
--review-history D=slow_query_log,t=global_query_review_history \
--no-report --limit=0% --filter=" \$event->{Bytes} = length(\$event->{arg}) and\$event->{hostname}=\"$HOSTNAME\"" /usr/local/mariadb/var/localhost-slow.log
endscript
```

### 3 多节点mySQL监控慢查询日志

node1:192.168.2.11   MariaDB10.0.17    还部署有nginx的anemometer web前端

node2:192.168.2.12  MariaDB10.0.17

各个节点的my.cnf里面开启慢查询，相关配置如下：

```javascript
[mysqld]
innodb_file_per_table = ON
skip_name_resolve = ON
slow_query_log=ON
slow_query_log_file =localhost-slow.log
long_query_time = 2
```

**1.** **安装anemometer**

node1上安装到nginx的网站目录下

```javascript
 cd /home/wwwroot/
 git clonehttps://github.com/box/Anemometer.git anemometer
 cd anemometer
```

node2上anemometer的安装目录没什么要求

```javascript
 cd /root
 git clone https://github.com/box/Anemometer.gitanemometer
 cd anemometer
```

**2.** **创建表和用户名**

node1上执行：

```javascript
# mysql -uroot -proot <install.sql
# mysql -uroot -proot -e"grant ALL ON slow_query_log.* to 'anemometer'@'192.168.2.%' IDENTIFIED BY'123456';"
# mysql -uroot -proot -e "grantSELECT ON *.* to 'anemometer'@'192.168.2.%' IDENTIFIED BY '123456';"
# mysql -uroot -proot -e"flush privileges;"
```

node2上执行：

```javascript
# mysql -uroot -proot <install.sql
# mysql -uroot -proot -e"grant ALL ON slow_query_log.* to 'anemometer'@'192.168.2.%' IDENTIFIED BY'123456';"
# mysql -uroot -proot -e"grant SELECT ON *.* to 'anemometer'@'192.168.2.%' IDENTIFIED BY'123456';"
# mysql -uroot -proot -e"flush privileges;"
```

**3.** **在两个节点执行pt命令分析慢查询日志，并写入到各自的数据库中**

node1上执行：

```javascript
# pt-query-digest --user=anemometer  --password=123456--host=192.168.2.11 \
--review h=192.168.2.11,D=slow_query_log,t=global_query_review\
--history h=192.168.2.11,D=slow_query_log,t=global_query_review_history\
--no-report --limit=0% --filter=" \$event->{Bytes} = length(\$event->{arg}) and \$event->{hostname}=\"$HOSTNAME\""localhost-slow.log
```

node2上执行：

```javascript
# pt-query-digest --user=anemometer  --password=123456--host=192.168.2.12 \
--review h=192.168.2.12,D=slow_query_log,t=global_query_review \
--history h=192.168.2.12,D=slow_query_log,t=global_query_review_history \
--no-report --limit=0% --filter=" \$event->{Bytes} = length(\$event->{arg}) and\$event->{hostname}=\"$HOSTNAME\"" localhost-slow.log
```

**4.** **在node1上配置前端**

```javascript
# cd /home/wwwroot/anemometer/conf
# cp sample.config.inc.php  config.inc.php
# vim config.inc.php  主要修改的地方如下2个【conf项，plugins项】：
$conf['datasources']['192.168.2.11'] = array(
        'host' => '192.168.2.11',
        'port' => 3306,
        'db'   => 'slow_query_log',
        'user' => 'anemometer',
        'password' => '123456',
        'tables' => array(
                'global_query_review' =>'fact',
                'global_query_review_history'=> 'dimension'
        ),
        'source_type' => 'slow_query_log'
);
 
$conf['datasources']['192.168.2.12'] = array(
        'host' => '192.168.2.12',
        'port' => 3306,
        'db'   => 'slow_query_log',
        'user' => 'anemometer',
        'password' => '123456',
        'tables' => array(
                'global_query_review' =>'fact',
                'global_query_review_history'=> 'dimension'
        ),
        'source_type' => 'slow_query_log'
);
 
$conf['plugins'] = array(
    ...省略代码...
       $conn['user'] = 'anemometer';
       $conn['password'] = '123456';
    ...省略代码...
# /etc/init.d/nginx restart   重启Nginx
```

Chrome查看http://192.168.2.11/ 如下图所示 

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/w7ci9j278k.png?imageView2/2/w/1620)

**5.** **下面是pt分析慢查询日志的脚本**

vim /home/scripts/pt-digest.sh 内容如下：

```javascript
#!/bin/bash
# 我这里直接把配置写死了，觉得不太好的话大家可以参考其它文章将数据库的连接配置独立出来
 
# 慢查询日志存放的目录
SQL_DATADIR="/usr/local/mariadb/var"
 
# 慢查询日志的文件名(basename)
SLOW_LOG_FILE=$( mysql -uroot -proot -e " show global variables like'slow_query_log_file'" | tail-n1 | awk '{ print $2 }' )
 
# 获取本机IP地址
IP_ADDR=$(/sbin/ifconfig | grep'inet addr'  | egrep '172.|192.' | awk'{print $2}' | awk -F ":" '{print $2}')
 
cp $SQL_DATADIR/$SLOW_LOG_FILE/tmp/
 
# 分析日志并存入slow_query_log这个数据库
/usr/local/bin/pt-query-digest --user=anemometer --password=123456 --host=$IP_ADDR \
 --review h=$IP_ADDR,D=slow_query_log,t=global_query_review\
 --history h=$IP_ADDR,D=slow_query_log,t=global_query_review_history\
 --no-report --limit=0% --filter="\$event->{Bytes} = length(\$event->{arg}) and\$event->{hostname}=\"$HOSTNAME\"" /tmp/$SLOW_LOG_FILE
 
rm -f /tmp/$SLOW_LOG_FILE
```

调试通过以后,在crontab添加如下命令实现定期采集慢查询日志到数据库存储

59 23 * * * /bin/bash /home/scripts/pt-digest.sh> /dev/null

这样每天就能自动分析采集慢查询日志了。

另外，慢查询日志建议按天切分，这样用pt-query-digest进行SQL慢查询日志统计的时候就避免重复分析了。慢查询按天切分的脚本如下：

**Tips下面是慢查询日志切分脚本:**

下面是一个轮询切割mySQL慢查询和错误日志的脚本（/home/scripts/mysql_log_rotate）：

```javascript
"/usr/local/mariadb/var/localhost-slow.log""/usr/local/mariadb/var/localhost_err" { 
    create 660 mariadb mariadb      # 这个文件权限和属主属组需要根据自己的情况修改
   dateext
   notifempty
   daily
   maxage 60
   rotate 30
   missingok
   olddir /usr/local/mariadb/var/oldlogs  # 这个目录不存在的话，要自己先新建好，并修改属主为mariadb
 
   postrotate
        if /usr/local/mariadb/bin/mysqladminping -uroot -proot &>/dev/null; then
            /usr/local/mariadb/bin/mysqladminflush-logs -uroot -proot
        fi
   endscript
}
```

![img](https://ask.qcloudimg.com/http-save/yehe-3663994/uw69apeacf.png?imageView2/2/w/1620)

**再配置个CRONTAB：**

```javascript
00 00 * * * (/usr/sbin/logrotate-f /home/scripts/mysql_log_rotate >/dev/null 2>&1)
```

 