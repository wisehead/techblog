---
title: 隔离级别、SI 和 SSI - 简书
category: default
tags: 
  - www.jianshu.com
created_at: 2020-09-12 16:55:41
original_url: https://www.jianshu.com/p/c348f68fecde
---

# 隔离级别、SI 和 SSI



# ACID

事务是关系数据库操作的**逻辑单位**。  
事务的存在，是为从数据库层面**保证数据的安全性，减轻应用程序的负担**。

说起“事务”，总会先想起 “**ACID**” 四个字母。

*   A：**A**tomicity，原子性。
*   C：**C**onsistency，一致性。
*   I：**I**solation，隔离性。
*   D：**D**urability，持久性。

原子性、一致性和持久性都比较好理解。

一个事务可能包含一个或多个操作，原子性保证这些操作**要么全部被生效，要么全部不被生效**。  
数据库的一致性是指数据库中的数据都**满足“完整性约束”**，如主键的唯一约束。

事务提交后，要**永久保存到数据库中**，这就是持久性。简单地说就是数据要落盘。为了提高系统的可用性，数据还应该通过某种算法复制到其它机器。

隔离性是这几个特性里面比较不好理解的。单个事务的场景下，谈隔离性是没意义的——事务之间有并发才有隔离的必要。简单地说，**隔离性指的就是数据库在并发事务下的表现**。权衡安全和性能，数据库一般会有多个隔离级别。

# 隔离级别

SQL 标准里定义了四个隔离级别：

*   **读未提交**（Read Uncommitted）：会出现**脏读**（Dirty Read）—— 一个事务会读到另一个事务的中间状态。
*   **读已提交**（Read Committed）：会出现**不可重复读**（Unrepeatable Read） —— 事务只会读到已提交的数据，但是一行数据读取两遍得到不同的结果。
*   **可重复读**（Repeatable Read）：会出现**幻读**（Phantom Read） —— 一个事务执行两个相同的查询语句，得到的是两个不同的结果集（数量不同）。
*   **可串行化**（Serializable）：可以找到一个事务串行执行的序列，其结果与事务并发执行的结果是一样的。

SQL 标准定义的的这四个隔离级别，只适用于基于锁的事务并发控制。后来有人写了一篇论文 _A Critique of ANSI SQL Isolation Levels_ 来批判 SQL 标准对隔离级别的定义，并在论文里提到了一种新的隔离级别 —— 快照隔离（Snapshot Isolation，简称 SI）。

# Snapshot Isolation

在 Snapshot Isolation 下，**不会**出现脏读、不可重复度和幻读三种读异常。并且读操作不会被阻塞，对于读多写少的应用 Snapshot Isolation 是非常好的选择。并且，在很多应用场景下，Snapshot Isolation 下的并发事务并不会导致数据异常。所以，主流数据库都实现了 Snapshot Isolation，比如 Oracle、SQL Server、PostgreSQL、TiDB、CockroachDB（关于 MySQL 的隔离级别，可以参考[这篇文章](https://www.jianshu.com/p/69fd2ca17cfd)）。

虽然大部分应用场景下，Snapshot Isolation 可以很好地运行，但是 Snapshot Isolation 依然没有达到可串行化的隔离级别，因为它会出现写偏序（write skew）。**Write skew 本质上是并发事务之间出现了读写冲突（读写冲突不一定会导致 write skew，但是发生 write skew 时肯定有读写冲突），但是 Snapshot Isolation 在事务提交时只检查了写写冲突。**

为了避免 write skew，应用程序必须根据具体的情况去做适配，比如使用`SELECT ... FOR UPDATE`，或者在应用层引入写写冲突。这样做相当于把数据库事务的一份工作扔给了应用层。

# Serializable Snapshot Isolation

后来，又有人提出了基于 Snapshot Isolation 的可串行化 —— Serializable Snapshot Isolation，简称 SSI（PostgreSQL 和 CockroachDB 已经支持 SSI）。

为了分析 Snapshot Isolation 下的事务调度可串行化问题，有论文提出了一种叫做 Dependency Serialization Graph (DSG) 的方法（可以参考下面提到的论文，没有深究原始出处）。通过分析事务之间的 rw、wr、ww 依赖关系，可以形成一个有向图。如果图中无环，说明这种情况下的事务调度顺序是可串行化的。这个算法理论上很完美，但是有一个很致命的缺点，就是复杂度比较高，难以用于工业生产环境。

_Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions_ 证明在 Snapshot Isolation 下, DSG 形成的环肯定有两条 rw-dependency 的边。

_Making snapshot isolation serializable_ 再进一步证明，这两条 rw-dependency 的边是“连续”的（一进一出）。

后来，_Serializable Isolation for snapshot database_ 在 Berkeley DB 的 Snapshot Isolation 之上，增加对事务 rw-dependency 的检测，当发现有两条“连续”的 rw-dependency 时，终止其中一个事务，以此避免出现不可串行化的可能。但是这个算法会有误判——不可以串行化的事务调用会出现两条“连续”的 rw-dependency 的边，但是出现两条“连续”的 rw-dependency 不一定会导致不可串行化。

_Serializable Snapshot Isolation in PostgreSQL_ 描述了上述算法在 PostgreSQL 中的实现。

上面提到的 Berkeley DB 和 PostgreSQL 的 SSI 实现都是单机的存储。_A Critique of Snapshot Isolation_ 描述了如何在分布式存储系统上实现 SSI，基本思想就是通过一个中心化的控制节点，对所有 rw-dependency 进行检查，有兴趣的可以参考论文。



