#1.创新点
##1.1 同步和崩溃恢复算法
Our  rst contribution is a replication and recovery algorithm that achieves high availability with a replication factor no higher than what is required for durability and without sacri cing performance or strong consistency guarantees. 

Log Store，1000个Log Server？？

##1.2.性能架构
Our second contribution is a set of novel architectural choices that enable Taurus to achieve higher performance. 
--200% higher throughput than MySQL 8.0
--Taurus supports multiple read replicas and keeps the replica lag under 20ms even under high load。
###存储节点优势
To avoid having the storage layer become a bottleneck, Taurus improves storage layer performance in two ways. First, the separation of Log Stores from Page Stores reduces the load on Page Stores. Second, the organization of Page Stores is optimized around a "the log is the database" model, never modifying data in place, and performing append-only writes. 

##1.3 详细介绍
The third contribution of this paper is a detailed description of the inner workings, performance optimizations, and design compromises in the storage layer. 

#2.特点
##2.1 架构
--1.SAL模块：相当于 PSM + brpc + RBIO Client + Log Send Client + 崩溃恢复。
--2.Log Store
--3.Page Store，没有用AFS。同时包括日志。
--4.主库，通过SAL写日志到log sotre + Page Store。

##2.2
--1.没有用AFS
--2.Page Store之间gossip，三个Page Server。
--3.每个Slice 10G，每个Page Store管理多个Slice。
--4.多个Log Server，负责Landing Zone。


#3.good idea for Page Server
--1.多版本读，可以日志落盘。解决log buffer撑爆的问题。
--2.page 可以落盘多版本，解决写放大的问题。
--3.从库buffer pool保存多版本。
--4.Page Store没有用AFS。

#4.疑问
--1.主库同时写log store和page store有什么好处？
--2.Page Store为什么是append only？？
never modifying data in place, and performing append-only writes.
Log structured storage was introduced by LFS [21] and has been applied in several databases and key-value stores [15, 20, 22, 25]. Taurus Page Stores use this concept to store its persistent data. 

--3.多版本如何实现？
--3.For each slice, there is a data structure called the Log Directory. It keeps track of the location of all log records and the versions of the pages hosted by the slice, i.e., information needed to produce pages.？？？
--4.The bu er pool functions as a write-back cache, asynchronously  ushing dirty pages to the slice log (step 6) allowing it to apply multiple log records before writing a page to disk, fur- ther reducing the amount of I/O. ？如何实现？落盘是否有额外的控制逻辑？？看着不像多版本？只有一个最新版本。
