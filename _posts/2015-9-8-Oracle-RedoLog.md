---
layout: post
title: ORACLE重做日志小结
category: oracle
excerpt: 关于redoLog的特点和优化（非DBA视角）
---
## 1.Redo log特点
-------------
*	重做日志以磁盘I/O为主，将数据库操作记录到日志文件。(磁盘I\O性能有可能成为瓶颈)
*	每个实例只有一个活动的LGWR(log writer)进程，至少有两个日志组(logfile group)，每个日志组至少一个	日志文件，同一日志组多个日志文件内容相同，互为镜像。
*	当一个日志组写满时，LGWR自动切换至下一个日志组，覆盖日志文件内容，继续写入。(频繁的日志切换可能影响性	能)
*	当切换到新的日志组时，如果DBWR(db writer)尚未将所有修改写入数据文件，LGWR将等待写入完成。(DBWR的	写入速度可能导致性能瓶颈)
*	增大日志文件可以减少日志切换频率，但会增加数据恢复的时间，同时，当日志文件损坏时，将会丢失更多的数据。
## 2.日志文件切换时间查询
-------------
```sql
Select a.sequence#, b.sequence#, a.first_time, b.first_time,
round((a.first_time-b.first_time)*60*60*24,2) timeInterval
from v$log_history a
left join v$log_history b
on a.sequence#=b.sequence#+1 and a.thread#=b.thread#
```
```
(Oracle建议日志文件切换时间为20分钟，调整日志文件大小以改变这一时间)
```
## 3.Redo log优化
-------------

*	同一日志组的日志文件分别放在不同的磁盘上，减少日志文件写入、日志切换时的竞争。
*	适当增大日志文件可以降低日志切换频率和LGWR等待DBWR的几率。
*	将日志文件放于RAID10或RAID0磁盘阵列中以提高写入性能。
*	增加DBWR进程数，提高数据写入性能。
## 4.创建、切换、删除日志文件组
-------------
调整日志文件大小只能通过，创建新的日志文件组，切换日志文件组，删除旧的日志文件组的方式进行。

创建日志文件组 

```sql
alter database add logfile group 4(‘D:\redo4.log’) size 1024M reuse;

```

查看日志文件组状态切换日志文件组 

```sql
select group#, status from v$log;

```

删除日志文件组(切换至INACTIVE状态再进行删除) 

```sql
alter database drop logfile group 1;

```
 