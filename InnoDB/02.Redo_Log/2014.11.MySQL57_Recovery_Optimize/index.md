---
title: MySQL · 5.7改进 · Recovery改进
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-09-05 15:53:55
original_url: http://mysql.taobao.org/monthly/2014/11/03/
---

[

# 数据库内核月报 － 2014 / 11

](http://mysql.taobao.org/monthly/2014/11)

[‹](http://mysql.taobao.org/monthly/2014/11/02/)

[›](http://mysql.taobao.org/monthly/2014/11/04/)

*   [当期文章](#)

## MySQL · 5.7改进 · Recovery改进

**背景**

InnoDB作为事务性引擎，使用write-ahead logging(WAL)机制保证ACID中的Atomicity和Durability，使用undo机制保证ACID中的Consistency和Isolation。

按照WAL和undo的机制，形成以下两个原则：

1.  数据块的更改需要先记录redo日志。
2.  数据块的更改需要先写入undo。 根据这两个原则，InnoDB更新数据的基本流程可以简单的总结为：
    
3.  记录需要更改undo record的redo log
4.  记录需要更改data record的redo log
5.  写入redo log
6.  写入undo record
7.  更新data record 这5个步骤。

**InnoDB Recovery**

如果MySQL实例异常crash，那么重启过程中首先会进行InnoDB recovery。 即：根据last checkpoint点，顺序读取后面的redo log，按照先前滚，再回滚的原则， 应用所有的redo log。

因为redo record中记录着数据块的地址(space\_id+page\_no)，所以recovery的过程首先会执行合并相同数据块的操作，以加快recovery的过程。

那么问题来了

根据space\_id怎么找到对应IDB数据文件？ 因为在恢复的过程中，InnoDB只load了redo文件和系统表空间文件，如何查找InnoDB的数据文件呢？

1.  InnoDB的数据字典dict\_table\_t结构中也保存了对应关系，但数据字典受redo保护，recovery的过程中不可用。
2.  扫描datadir的所有数据文件，读取page中保存的space\_id，建立space\_id和数据文件的对应关系。 MySQL目前采用第二种方式，但带来了一个问题，当设置了innodb\_file\_per\_table后，每一个表对应一个表空间，那么需要读取所有的目录下的所有Innodb数据文件，这样就会严重的影响了recovery的时间。

MySQL 5.7改进策略：

MySQL 5.7中，在redo log中增加了一种新的record类型，即MLOG\_FILE\_NAME，记录了自last checkpoint以来更改的数据文件的file name。 这样在应用的时候，直接根据文件名就可以找到数据文件。

Oracle的设计机制：

Oracle数据库recovery的过程中，有没有这个问题呢？ 答案是没有。

我们来看下Oracle的设计机制:

oracle同样在系统表空间中记录了数据字典，受redo保护，可以通过DBA\_开头的表来查询。但Oracle还维护了一个control file，控制文件中记录了database name，redo file，datafile，backup等信息，通过v$开头的表查询。 当Oracle在recovery的过程中，需要数据库在mount状态下，即打开了控制文件，这时数据字典还不可用(DB没有open)，在应用redo log的时候，根据控制文件中的v$datafile，检索file\_id和file\_name的对应关系。 MySQL是根据datadir，innodb\_data\_home\_dir，innodb\_log\_group\_home\_dir等几个目录配置，通过文件系统的查找，找到相应文件的，而Oracle维护了一个集中式的control file管理这些初始加载的文件地址。

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1599292435-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/11/03/)

创建于: 2020-09-05 15:53:55

目录: default

标签: `mysql.taobao.org`

