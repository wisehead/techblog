---
title: MySQL · 性能优化 · Bulk Load for CREATE INDEX
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-09-05 16:30:47
original_url: http://mysql.taobao.org/monthly/2014/12/07/
---

[

# 数据库内核月报 － 2014 / 12

](http://mysql.taobao.org/monthly/2014/12)

[‹](http://mysql.taobao.org/monthly/2014/12/06/)

[›](http://mysql.taobao.org/monthly/2014/12/08/)

*   [当期文章](#)

## MySQL · 性能优化 · Bulk Load for CREATE INDEX

**背景**

MySQL5.6以后的版本提供了多种优化手段用于create index，比如online方式，Bulk Load方式。

Online提供了非阻塞写入的方式创建索引，为运维提供了很大的便利。 Bulk Load提升了索引创建的效率,减少了阻塞的时间。 这篇介绍下MySQL 5.7.5 Bulk Load的细节，并从查找，排序，redo，undo，page split等维度比较一下传统方式和Bulk Load方式。

**传统方式**

MySQL 5.7.5版本之前，create index使用的是和insert一条记录相同的api接口，即自上而下的插入方式。

步骤1: 扫描clustered index，得到行记录。 步骤2: 根据record，按照B-Tree的结构，从root->branch->leaf page查找到属于record的位置。: 步骤3: 调用write index record接口，维护索引。

1.  查找: 对每一条记录在插入前从B-Tree上查找到自己的位置。
2.  排序: 因为是按照B-Tree的结构，所以每一条记录插入都是有序的。
3.  redo: 每条记录的插入都会记录innodb的redo做保护。
4.  undo: 记录每个插入记录位置的undo
5.  page split: 插入采用optimistic的方式，如果失败而发现page full，那么就split page，并向上更新branch page。

从上面的步骤和几个维度的说明上，传统的create index比较简单，但一方面会阻塞写入，另一方面效率会比较低，延长了不可用时间。

**Bulk Load方式**

MySQL 5.7.5 版本，提供了Bulk Load方式创建索引，使用多路归并排序和批量写入的方法，是一种自下而上的方式。

步骤1: 扫描clustered index，写入sort buffer，等sort buffer写满了后，写入到临时文件中。 步骤2: 对临时文件中的有序记录进行归并排序。 步骤3: 批量写入到索引结构中。

批量写入: 因为记录都是有序的，所以写入的过程就是，不断的分配leaf page，然后批量写入记录，并保留innodb\_fill\_factor设置的空闲空间大小，所以，就是不断在最右边leaf page写入，并不断进行平衡B-Tree结构的过程。

1.  查找: Bulk Load方式并没有单条record查找的过程。
2.  排序: 使用多路归并排序，对待写入的records进行排序。
3.  redo: Innodb并没有记录redo log，而是做checkpoint直接持久化数据。
4.  undo: 记录了新分配的page。
5.  page split: 因为每次都是初始化一个最右端的page，create index的时候不存在split。

从上面的步骤和几个维度的说明上，Bulk Load方式能显著的利用机器的吞吐量，加快创建index的过程。

**问题及与Oracle的比较**

1.  临时空间使用
    
    MySQL使用临时目录来保存临时文件，对于文件的大小受限于目录空间大小，需要注意。RDS通过增加一个参数来控制临时空间的使用。 Oracle使用临时表空间，如果排序空间不足，则会遇到常见的错误：ORA-01652: unable to extend temp segment by 128 in tablespace TEMP
    
2.  redo保护
    
    MySQL 的Bulk Load方式，没有使用redo保护，数据库从write-ahead logging方式退化成direct persist data，并且未来如果MySQL希望使用Innodb redo的方式进行复制，也变的困难。 Oracle如果不指定no logging参数，索引创建过程中记录完整的redo信息。
    
3.  direct write
    
    MySQL Bulk Load方式，对于新的leaf page，在创建的过程中，唤醒page cleaner线程对这些page做checkpoint进行持久化。 Oracle提供一种用户的服务器进程直接direct write物理文件的方式，写入数据，而不依赖DBWR进程。
    

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1599294647-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/12/07/)

创建于: 2020-09-05 16:30:47

目录: default

标签: `mysql.taobao.org`

