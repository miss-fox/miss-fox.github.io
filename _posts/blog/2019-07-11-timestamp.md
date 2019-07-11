---
layout: post
title: timestamp
description: 数据库时间类型对比
category: blog
---


>今天一个同事在navicat上通过时间戳过滤查询日志，发现where没有起到作用（也给我吓一身汗，以为代码有bug），后面发现field的type是 timestamp . 之前也不太了解这个type,就查了一下。下面是当时查询的sql对比

```mysql
select `action`,count(*) from table1 where type=13101 and created_at>1561420800 group by `action` order by `action`;//error
select `action`,count(*) from table1 where type=13101 and created_at>'2019-06-25 00:00:00' group by `action` order by `action`;//correct
```

   1. Timestamp类型虽然每次查询出来都是形如'2019-06-25 00:00:00'的格式，其实 底层存储的时间戳。展示的值依赖时区。四个字节，只能到2038年。

   2. 时区：
      Client连接数据库的时候会有timezone设定，连接过来的是timeformat（php代码string）的数据，存储的时候，会根据连接的时区转换成timestamp存入。这条数据在不同时区的客户端的展示可能会不同。client-connection的 default值参考[mysql文档](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_time_zone)，默认是global timezone（而global 的init是system timezone）。也可以通过SET time_zone='+04:00';的方式重新设置当前 会话的时区。demo如下：

      表结构 ：实际存储的是1562840484这个时间戳（北京的18:21:24）
   
      ```mysql
      //表结构
      CREATE TABLE `yytest` (
        `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
        `test_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
        `test_date` datetime DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
      ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COMMENT='email信息'
      ```
   
      ```php
      ///////代码
      $dbh = new PDO('mysql:host=127.0.0.1:3306;dbname=mysql', 'root', '123');
      $dbh->exec("SET time_zone='+04:00';");
      $tt = $dbh->query("show variables like '%time_zone%';");
      foreach ($tt as $tl) {
        var_dump($tl);//打印结果是+04:00
      }
      $sql = "INSERT INTO yytest(test_time) VALUES ('2019-07-11 14:21:24')";
      $result = $dbh->exec($sql);
      foreach($dbh->query('SELECT * from yytest') as $row) {
        print_r($row);
      }
      
      ///output
      @client1
       Array
       (
           [id] => 37
           [0] => 37
           [test_time] => 2019-07-11 14:21:24
           [1] => 2019-07-11 14:21:24//东四区的读取
           [test_date] => 2019-07-11 19:48:24
           [2] => 2019-07-11 19:48:24//不变
       )
      ```
   
      
   
      对应的本地的navicat查看的结果是：本地的是utc+8（中国时区）@client2
   
      ```php
      test_time 2019-07-11 18:21:24//东八区的读取
      test_date 2019-07-11 19:48:24//不变
      ```
   
      可以看出client1通过代码强制把时区设置了东四区，navicate是东八区，两个客户端读到的test_time展示不同。    同时也发现对于datetime类型底层存的就是文本，没有时区转换的概念。
   
2. timestamp客户端时区不一致的问题

   我们线上的机器的mysql的系统timezone设置是Utc，而业务层的时区是PRC（mysql的连接的时区跟业务层没关系，这个地方之前我就搞混了）,下面本地模拟一下

   ```php
   date_default_timezone_set('PRC');
   $dbh = new PDO('mysql:host=127.0.0.1:3306;dbname=mysql', 'root', '123');
   $dbh->exec("SET time_zone='+00:00';");//线上数据库的时区是utc，默认连接也是utc，所以这个地方通过设置会话的时区模拟线上的情况，当然也可以把本地数据库的时区改成utc来模拟，懒得改配置重启了。
   foreach($dbh->query('SELECT * from yytest') as $row) {
       print_r($row);
       var_dump(strtotime($row['test_time']));
   }
   ```

   ```php
   Array
   (
       [id] => 51
       [0] => 51
   [test_time] => 2019-07-11 10:21:24//这里会话是utc时区，所以对比东四的慢4个小时
       [1] => 2019-07-11 10:21:24
       [test_date] => 2019-07-11 19:48:24
       [2] => 2019-07-11 19:48:24
   )
   int(1562811684)//1562840484这个是一开始写入的时间戳
   
   ```

   这里php的代码里是prc时区，对于数据库返回的test_time（string），再做转化的时候就不对了，所以尽量保证两端的时区一致，这样不容易出差错。

3. Timestamp,datetime,int

   平时写入的时候一般会用int存储时间戳，这样代码层面根据自己的时区随便转化，可以避免2提到的问题，不用timestamp,客户端读取的时候更直观。timestamp占4个字节（最大,2038/1/19）

   datetime存储的是文本。占用8个字节。

   [性能](https://www.jianshu.com/p/b22ac1754372)









