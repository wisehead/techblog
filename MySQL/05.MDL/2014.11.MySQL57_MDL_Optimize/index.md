---
title: MySQL · 5.7优化 · Metadata Lock子系统的优化
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-09-05 15:29:59
original_url: http://mysql.taobao.org/monthly/2014/11/05/
---

[

# 数据库内核月报 － 2014 / 11

](http://mysql.taobao.org/monthly/2014/11)

[‹](http://mysql.taobao.org/monthly/2014/11/04/)

[›](http://mysql.taobao.org/monthly/2014/11/06/)

*   [当期文章](#)

## MySQL · 5.7优化 · Metadata Lock子系统的优化

**背景**

引入MDL锁的目的，最初是为了解决著名的bug#989，在MySQL 5.1及之前的版本，事务执行过程中并不维护涉及到的所有表的Metatdata 锁，极易出现复制中断，例如如下执行序列：

Session 1: BEGIN; Session 1: INSERT INTO t1 VALUES (1); Session 2: Drop table t1; ——–SQL写入BINLOG Session 1: COMMIT; —–事务写入BINLOG 在备库重放 binlog时，会先执行DROP TABLE，再INSERT数据，从而导致复制中断。

在MySQL 5.5版本里，引入了MDL, 在事务过程中涉及到的所有表的MDL锁，直到事务结束才释放。这意味着上述序列的DROP TABLE 操作将被Session 1阻塞住直到其提交。

不过用过5.5的人都知道，MDL实在是个让人讨厌的东西，相信不少人肯定遇到过在使用mysqldump做逻辑备份时，由于需要执行FLUSH TABLES WITH READ LOCK (以下用FTWRL缩写代替)来获取全局GLOBAL的MDL锁，因此经常可以看到“wait for global read lock”之类的信息。如果备库存在大查询，或者复制线程正在执行比较漫长的DDL，并且FTWRL被block住，那么随后的QUERY都会被block住，导致业务不可用引发故障。

为了解决这个问题，Facebook为MySQL增加新的接口替换掉FTWRL 只创建一个read view ，并返回与read view一致的binlog位点；另外Percona Server也实现了一种类似的办法来绕过FTWRL，具体点击文档连接以及percona的博客，不展开阐述。

MDL解决了bug#989，却引入了一个新的热点，所有的MDL锁对象被维护在一个hash对象中；对于热点，最正常的想法当然是对其进行分区来分散热点，不过这也是Facebook的大神Mark Callaghan在report了bug#66473后才加入的，当时Mark观察到MDL\_map::mutex的锁竞争非常高，进而推动官方改变。因此在MySQL 5.6.8及之后的版本中，引入了新参数metadata\_locks\_hash\_instances来控制对mdl hash的分区数(Rev:4350)；

不过故事还没结束，后面的测试又发现哈希函数有问题，类似somedb.someprefix1….somedb.someprefix8的hash key值相同，都被hash到同一个桶下面了，相当于hash分区没生效。这属于hash算法的问题，喜欢考古的同学可以阅读下bug#66473后面Dmitry Lenev的分析。

Mark进一步的测试发现Innodb的hash计算算法比my\_hash\_sort\_bin要更高效， Oracle的开发人员重开了个bug#68487来跟踪该问题，并在MySQL5.6.15对hash key计算函数进行优化，包括fix 上面说的hash计算问题(Rev:5459)，使用MurmurHash3算法来计算mdl key的hash值。

**MySQL 5.7 对MDL锁的优化**

在MySQL 5.7里对MDL子系统做了更为彻底的优化。主要从以下几点出发：

第一，尽管对MDL HASH进行了分区，但由于是以表名+库名的方式作为key值进行分区，如果查询或者DML都集中在同一张表上，就会hash到相同的分区，引起明显的MDL HASH上的锁竞争

针对这一点，引入了LOCK-FREE的HASH来存储MDL\_lock，LF\_HASH无锁算法基于论文”Split-Ordered Lists: Lock-Free Extensible Hash Tables”，实现还比较复杂。 注：实际上LF\_HASH很早就被应用于Performance Schema，算是比较成熟的代码模块。

由于引入了LF\_HASH，MDL HASH分区特性自然直接被废除了 。

对应WL#7305， PATCH(Rev:7249)

第二，从广泛使用的实际场景来看，DML/SELECT相比DDL等高级别MDL锁类型，是更为普遍的，因此可以针对性的降低DML和SELECT操作的MDL开销。

为了实现对DML/SELECT的快速加锁，使用了类似LOCK-WORD的加锁方式，称之为FAST-PATH，如果FAST-PATH加锁失败，则走SLOW-PATH来进行加锁。

每个MDL锁对象（MDL\_lock）都维持了一个long long类型的状态值来标示当前的加锁状态，变量名为MDL\_lock::m\_fast\_path\_state 举个简单的例子：（初始在sbtest1表上对应MDL\_lock::m\_fast\_path\_state值为0）

Session 1: BEGIN; Session 1: SELECT \* FROM sbtest1 WHERE id =1; //m\_fast\_path\_state = 1048576, MDL ticket 不加MDL\_lock::m\_granted队列 Session 2: BEGIN; Session 2: SELECT \* FROM sbtest1 WHERE id =2; //m\_fast\_path\_state=1048576+1048576=2097152，同上，走FAST PATH Session 3: ALTER TABLE sbtest1 ENGINE = INNODB; //DDL请求加的MDL\_SHARED\_UPGRADABLE类型锁被视为unobtrusive lock，可以认为这个是比上述SQL的MDL锁级别更高的锁，并且不相容，因此被强制走slow path。而slow path是需要加MDL\_lock::m\_rwlock的写锁。m\_fast\_path\_state = m\_fast\_path\_state | MDL\_lock::HAS\_SLOW\_PATH | MDL\_lock::HAS\_OBTRUSIVE 注:DDL还会获得库级别的意向排他MDL锁或者表级别的共享可升级锁，但为了表述方便，这里直接忽略了，只考虑涉及的同一个MDL\_lock锁对象。 Session 4: SELECT \* FROM sbtest1 WHERE id =3; // 检查m\_fast\_path\_state &HAS\_OBTRUSIVE，如果DDL还没跑完，就会走slow path。 从上面的描述可以看出，MDL子系统显式的对锁类型进行了区分（OBTRUSIVE or UNOBTRUSIVE），存储在数组矩阵m\_unobtrusive\_lock\_increment。 因此对于相容类型的MDL锁类型，例如DML/SELECT，加锁操作几乎没有任何读写锁或MUTEX开销。

对应WL#7304, WL#7306 ， PATCH（Rev:7067,Rev:7129）(Rev:7586)

第三，由于引入了MDL锁，实际上早期版本用于控制Server和引擎层表级并发的THR\_LOCK 对于Innodb而言已经有些冗余了，因此Innodb表完全可以忽略这部分的开销。

不过在已有的逻辑中，Innodb依然依赖THR\_LOCK来实现LOCK TABLE tbname READ，因此增加了新的MDL锁类型来代替这种实现。

实际上代码的大部分修改都是为了处理新的MDL类型，Innodb的改动只有几行代码。

对应WL#6671，PATCH(Rev:8232)

第四，Server层的用户锁（通过GET\_LOCK函数获取）使用MDL来重新实现。

用户可以通过GET\_LOCK()来同时获取多个用户锁，同时由于使用MDL来实现，可以借助MDL子系统实现死锁的检测。

注意由于该变化，导致用户锁的命名必须小于64字节，这是受MDL子系统的限制导致。

对应WL#1159, PATCH(Rev:8356)

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1599290999-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/11/05/)

创建于: 2020-09-05 15:29:59

目录: default

标签: `mysql.taobao.org`

