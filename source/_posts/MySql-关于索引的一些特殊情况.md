title: MySql 关于索引的一些特殊情况
author: Peilin Deng
tags:
  - MySql
categories:
  - 摘抄笔记
date: 2021-08-25 21:59:00
---
# 范围查询数据量过大导致索引失效
存在索引idx_fk_customer_id(customer_id)，表中数据16000条。
```sql
EXPLAIN SELECT * FROM rental WHERE customer_id<102; # 使用索引
EXPLAIN SELECT * FROM rental WHERE customer_id<103; # 全表扫描
```
当范围查询时，如果符合条件的数据过多时，因为建立索引的字段虽然在索引树上有序，但是这一部分数据还要回源到聚簇索引中再次查询，并且得到的数据在磁盘上并不是连续的，这样会产生大量的随机IO，而随机IO是非常慢的，与其这样还不如全表扫描。全表扫描最起码是顺序IO。

# Semi join半连接





+ explain列中filtered为什么有时候不准
+ Extra列中Using index condition都是索引下推吗？