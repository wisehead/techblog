---
title: MariaDB · 特性分析· Per-query variables
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-10-14 16:29:27
original_url: http://mysql.taobao.org/monthly/2015/02/09/
---


# MariaDB · 特性分析· Per-query variables

自MariaDB 10.1.2起，MariaDB提供了一种"Per-query variables的方式来为Query设置语句级变量，通过 SET STATEMENT 语句可以为接下来要执行的语句设置一些系统变量值。

**语法**

SET STATEMENT var1=value1 \[, var2=value2, …\] FOR

varN是一个系统变量，valueN是一个常量值。但是有部分变量是不支持的，在这个章节的末尾列出了所有不支持的变量。

一条 "SET STATEMENT var1=value1 FOR stmt" 语句等价与如下一系列操作：

SET @save\_value=@@var1;

SET SESSION var1=value1;

stmt;

SET SESSION var1=@save\_value;

MySQL服务器在执行整条语句前会先做解析，所以所有影响解析器的变量都无法达到预期的效果，因为解析完之后才能获得这些变量的值。例如字符集变量sql\_mode=ansi\_quotes。

**一些使用特性的例子**

可以限制语句的执行时间 max\_statement\_time: SET STATEMENT max\_statement\_time=1000 FOR SELECT … ;

为一个语句临时改变优化器的规则: SET STATEMENT optimizer\_switch='materialization=off' FOR SELECT ….;

为一个语句单独打开MRR/BKA特性: SET STATEMENT join\_cache\_level=6, optimizer\_switch='mrr=on' FOR SELECT …

**下面这些变量无法使用Per-query variables特性来设置**

autocommit

character\_set\_client

character\_set\_connection

character\_set\_filesystem

collation\_connection

default\_master\_connection

debug\_sync

interactive\_timeout

gtid\_domain\_id

last\_insert\_id

log\_slow\_filter

log\_slow\_rate\_limit

log\_slow\_verbosity

long\_query\_time

min\_examined\_row\_limit

profiling

profiling\_history\_size

query\_cache\_type

rand\_seed1

rand\_seed2

skip\_replication

slow\_query\_log

sql\_log\_off

tx\_isolation

wait\_timeout

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2015/02/09/)

创建于: 2020-10-14 16:29:27

目录: default

标签: `mysql.taobao.org`

