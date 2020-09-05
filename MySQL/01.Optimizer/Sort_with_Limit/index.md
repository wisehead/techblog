---
title: MariaDB · 性能优化 · filesort with small LIMIT optimization
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-09-05 14:45:11
original_url: http://mysql.taobao.org/monthly/2014/11/10/
---

[

# 数据库内核月报 － 2014 / 11

](http://mysql.taobao.org/monthly/2014/11)

[‹](http://mysql.taobao.org/monthly/2014/11/09/)

*   [当期文章](#)

## MariaDB · 性能优化 · filesort with small LIMIT optimization

从MySQL 5.6.2/MariaDB 10.0.0版本开始，MySQL/MariaDB针对”ORDER BY …LIMIT n”语句实现了一种新的优化策略。当n足够小的时候，优化器会采用一个容积为n的优先队列来进行排序，而不是排序所有数据然后取出前n条。 这个新算法可以这么描述：（假设是ASC排序）

1.  建立一个只有n个元素的优先队列（堆），根节点为堆中最大元素
2.  根据其他条件，依次从表中取出一行数据
3.  如果当前行的排序关键字小于堆头，则把当前元素替换堆头，重新Shift保持堆的特性
4.  再取一条数据重复2步骤，如果没有下一条数据则执行5
5.  依次取出堆中的元素（从大到小排序），逆序输出（从小到大排序），即可得ASC的排序结果

这样的算法，时间复杂度为m_log(n)，m为索引过滤后的行数，n为LIMIT的行数。而原始的全排序算法，时间复杂度为m_log(m)。只要n远小于m，这个算法就会很有效。

不过在MySQL 5.6中，除了optimizer\_trace，没有好的方法来看到这个新的执行计划到底起了多少作用。MariaDB 10.013开始，提供一个系统状态，可以查看新执行计划调用的次数：

```makefile
Sort_priority_queue_sorts
描述: 通过优先队列实现排序的次数。(总排序次数=Sort_range+Sort_scan)
范围: Global, Session
数据类型: numeric
引入版本: MariaDB 10.0.13
```

此外，MariaDB还将此信息打入了Slow Log中。只要指定 log\_slow\_verbosity=query\_plan，就可以在Slow Log中看到这样的记录：

```sql
# Time: 140714 18:30:39
# User@Host: root[root] @ localhost []
# Thread_id: 3  Schema: test  QC_hit: No
# Query_time: 0.053857  Lock_time: 0.000188  Rows_sent: 11  Rows_examined: 100011
# Full_scan: Yes  Full_join: No  Tmp_table: No  Tmp_table_on_disk: No
# Filesort: Yes  Filesort_on_disk: No  Merge_passes: 0  Priority_queue: Yes
SET timestamp=1405348239;SET timestamp=1405348239;
select * from t1 where col1 between 10 and 20 order by col2 limit 100;
```

“Priority\_queue: Yes” 就表示这个Query利用了优先队列的执行计划(pt-query-digest 目前已经可以解析 Priority\_queue 这个列)。

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1599288311-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/11/10/)

创建于: 2020-09-05 14:45:11

目录: default

标签: `mysql.taobao.org`

