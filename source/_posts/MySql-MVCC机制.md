title: MySql MVCC机制
author: Peilin Deng
tags:
  - MySql
categories:
  - 摘抄笔记
date: 2021-08-25 21:21:00
---
> MVCC：Multi-Version Concurrent Control，多版本并发控制。
# 前情提要
1. 当多个线程执行事务的时候，对同一个缓存页里的一行数据进行更新。这个冲突如何处理
2. 当一个事务更新一条数据时，另一个事务在查询这条数据。

# 常见问题
## 脏写&脏读
因为一个事务去更新或者查询了另一个没有提交的事务更新过去的数据。因为另一个事务还没提交，所以随时可能回滚。导致自己更新的数据或者查询的数据没了。

## 不可重复读
在一个事务开始之后，多次读取同一条数据的结果因为其他事务修改的提交，显示为多次读取到不同的值。

## 幻读
<!-- more -->
在一个是事务开始后，多次读取一组数据的结果因为其他事务的新增或者删除的提交，显示为新增了数据或者减少了数据。

# 四种隔离级别
## 读未提交
read uncommitted    
能够解决脏写，因为一个事务对同一条数据进行操作时（更新，删除），其他对该条数据的操作的事务将会卡住。当第一个事务提交后第二个事务才会执行。否则第二个事务等待一段时间后报错。一般没人用这个。   
![](/images/img-42.png)
![](/images/img-43.png)

## 读已提交
read committed    
能够解决脏读和脏写，只会读取到其他事物已经提交的数据。ORACLE的默认隔离级别
## 可重复读
repeatable read 
能够解决脏读，脏写和不可重复读。但是会出现幻读。事务一旦开始，多次查询一个值，会一直读取到同一个值。*MYSQL的默认隔离级别，MYSQL中RR级别不存在幻读。*
## 串行化
serializable    
基本解决以上所有问题，因为事务是串行进行，不存在并发的情况。

## 隔离级别的修改
set [global | session] transaction_isolation level xxx    
xxx：REPEATABLE READ，READ COMMITTED，READ UNCOMMITTED，SERIALIZABLE

# 基于undo多版本链表实现的ReadView机制
<font color="red">在MYSQL中，事务的ID是自增的，这是一个重点！！！</font>    
在执行一个事务的时候，会生成一个ReadView，其中包括了以下4个（不仅仅是4个）关键内容：
+ m_ids：在生成readview时，当前系统中==活跃的读写事务==的事务id列表【当前事务（新建事务）与正在内存中commit的事务不在活跃事务列表中】
+ min_trx_id：表示在生成readview时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值
+ max_trx_id：生成readview时系统中==应该分配给下一个事务的id值==
+ creator_trx_id：生成readview的事务的事务id    
通过这些信息，在Readview生成后，当前事务更新的数据可以被自身读到，或者是在Readview生成前提交的事务修改的值，也是可以读到的。但是在生成ReadView的时候就已经活跃的事务，或者是ReadView生成后再开启的事务，修改的数据也是读不到的。

## 举个栗子（RR）
0. 数据库中存在一条数据，事务id是32，是初始值。  
![](/images/img-44.png)

1. 事务A（trx=36）和事务B（trx=39）同时开启，事务A查询，事务B更新。
![](/images/img-46.png)

2. 事务A根据min_trx=36>trx_id=32，知道这个数据在事务开启前就已经提交过了。所以可以查询到该条数据。    
事务B同理也能查询到，然后事务B把值改成了值B。
![](/images/img-47.png)

3. 事务A再次查询的时候，此时数据中的trx_id=39处于[min_trx_id,max_trx_id]，说明更新数据的事务是和自己差不多同时开启的，并且trx_id=39∈[36,39]。所以就不能查询这条数据了

4. 虽然事务A不能查询trx_id=39这条数据，但是可以顺着roll_ptr找下去，能找到最近的一条trx_id=32<36的数据。说明这一个undo log版本是在事务A开启之前就提交过的。所以查询得到的是原始值。
![](/images/img-48.png)

5. 如果此时事务A更新该数据为值A，trx_id修改为36，同时保存之前事务B修改的值的快照，如下图
![](/images/img-49.png)

6. 当事务A来查询数据时，发现trx_id=45刚好和自己Read_View中的creator_trx_id一致，说明数据是自己修改的，自己可以直接读取到。

7. 当事务C来进行一次更新操作后。事务A再去读取，发现trx_id=41>max_trx_id，说明在自己开启事务后有一个事务去更新了这笔数据，自己也不能去查询。然后顺着roll_ptr刚好找到一个trx_id=36的undo日志，说明是自己修改过的，直接拿到了值A
![](/images/img-50.png)

## READ COMMITTED是如何基于READ VIEW实现的
核心点在于：每次发起的查询，都生成一个新的Read View    
上面的例子中，第2步事务B提交后，事务A查询时，创建新的Read View，此时活跃的事务就只有事务A（trx_id=36）了，查询到的数据trx_id=39，在min_trx_id和max_trx_id这个区间内，并且不在m_ids列表中，说明已经提交了，所以可以读取到这个值。    
![](/images/img-51.png)

## REPEATABLE READ是如何基于READ VIEW实现的
核心点在于：每次发起的查询，使用的Read View仍然是第一次SELECT生成的。所以即便其他的事务提交了，m_ids中的内容也不会发生变化。    

## RR如何基于READ VIEW解决幻读
在事务A开启后，进行了一次范围查询；之后事务C插入了符合范围查询的数据，但是这些数据的DB_TRX_ID是事务C的ID，因为Read View只会创建一次，事务C大于事务A的ReadView中的max_trx_id，所以事务A的再次查询是获取不到新增的数据的。

A57