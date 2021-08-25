title: MySql Redolog（重做日志）
author: Peilin Deng
tags:
  - MySql
categories:
  - 摘抄笔记
date: 2021-08-25 21:11:00
---
# 前情提要
在执行增删改操作的时候，是基于Buffer Pool中的缓存页中的数据的。更新了缓存页中的数据后，会写入一条数据到Redo Log中。在提交事务的时候，会立马把Redo Log刷入磁盘（推荐方式），然后在RedoLog中写入binlog信息和一个commit标记，事务至此提交完毕。

# FAQ
## RedoLog保障了什么
当更新了缓存中的数据页后，缓存页还没有刷到磁盘上。MYSQL宕机，那么MYSQL重启后，会直接读取RedoLog中的内容，重做到Buffer Pool中，然后在刷到磁盘。因为RedoLog是顺序读写，所以效率很高。并且为了保证数据的不丢失，RedoLog也要设置成不经过OS Cache。

## 为什么要写入RedoLog，直接刷到磁盘的缺点在哪里？
1. 一个缓存页是16KB，刷盘比较耗时。而且你可能只修改了几个字节的数据。
2. 缓存页刷入磁盘是随机读写。效率很低。而RedoLog是顺序读写，效率高

<!-- more -->

# 详细介绍
## RedoLog的类型
根据数据页修改的字节数划分了不同的类型    
+ MLOG_1BYTE：修改了1个字节
+ MLOG_2BYTE：修改了2个字节
+ MLOG_4BYTE：修改了4个字节
+ MLOG_WRITE_STRING：修改了一大串的值

## RedoLog的大致内容
MLOG_NBYTE，表空间ID，数据页号，数据页中的偏移量，具体修改的值    
MLOG_WRITE_STRING，表空间ID，数据页号，数据页中的偏移量，修改的长度，具体修改的值

## RedoLog Block
RedoLog中，包含着多个RedoLog Block。每个RedoLog大小是512KB；    
在写入Redolog之前先写入内存中的RedoLog Block，然后再把RedoLog Block写入磁盘文件。具体的结构见图。    
![](/images/img-40.png)

## RedoLog buffer
好比Buffer Pool，基于内存中的一块连续空间。里面分配了多个空的RedoLog block。用来存放redolog。    
+ 在一个事务中，多个操作对应多个redo Log，对应着一组redolog，现在别的地方暂存，执行完了再把一组的redolog写入到内存中的redolog buffer中的block中
+ 如果一个事务对应的redoLog太多，就会放到两个甚至多个redolog block中。
+ 如果一个redolog group比较小，也可能会把多个redolog group合并在一个redolog block中。

## 刷盘时机
1. 写入redolog buffer的日志占用了总容量（innodb_log_buffer_size）的50%。
2. 事务提交的时候
3. 后台线程定时刷新
4. MYSQL关闭的时候

## RedoLog的一些默认处理
磁盘上默认redolog数量：2个(innodb_log_files_in_group)    
磁盘上默认redolog大小：48MB(innodb_log_file_size)    
磁盘上的默认名：ib_logfile0，ib_logfile1    
一般情况是向一个redolog中写，写满了就换下一个。所以redolog最多保存96MB的redolog，如果第二个写满了，就覆盖第一个日志文件里面原来的redolog