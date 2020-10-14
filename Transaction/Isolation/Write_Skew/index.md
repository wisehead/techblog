---
title: mysql的write skew问题 - 程序园
category: default
tags: 
  - www.voidcn.com
created_at: 2020-09-12 17:07:30
original_url: http://www.voidcn.com/article/p-zwgnfnqs-da.html
---

# mysql的write skew问题


测试环境版本5.5.50-MariaDB，存储引擎innodb；数据库实例test，用户root；  
关闭自动提交；  
set autocommit=0;  
查看事物隔离级别为RR：select @@global.tx\_isolation,@@tx\_isolation;  
创建测试表和数据  
create table worker1(n int primary key);  
create table worker2(n int primary key);  
create table user(userid varchar32 primary key,count int);  
insert into worker1 values(0);  
insert into worker2 values(0);  
insert into user values('001',0);  
commit;  
  
第一个问题，write skew（直译写偏）；  
开启两个mysql cli  
1)第一个终端 select \* from worker1;此时n位0；  
2)第二个终端 update worker1 set n = n+1;select \* from worker1;commit;此时n为1；  
3)第一个终端，再次执行select \* from worker1;发现n为0；  
  执行update worker1 set n = n+1;select \* from worker1;commit;  
  发现update语句执行后，worker1表变为的2调记录，一个n为0，一个n为2；commit之后，worker1表只有一条记录n为2；  
问题，说明第一个终端的事物中，查询到的是老版本的n值，但是执行update语句，update的是n的新版本，未提交前，新老版本的记录都可见，因此repeatable read实现的是有问题的；  
  
修改事物隔离级别为serializable后该问题不存在：  
set global transaction isolation level  SERIALIZABLE;  

修改后，第一个终端执行select读不提交，第二个终端执行update操作会一直阻塞，如果第一个读事物不提交，第二个事物会超时失败。

后记，关于write skew的说法，下面链接有一个详细的说明。似乎跟本例中的例子还不太一样。

从参考上看，不同的数据库其MVCC事务引擎对于write skew的表现相差很大。

结合以前的资料，PG的serializabe的实现比较特别，叫做serializable snapshot isolation。mysql的serializable是基于悲观锁的2PL协议实现的。  

https://vladmihalcea.com/2015/10/20/a-beginners-guide-to-read-and-write-skew-phenomena/

