# 数据并发问题

| sno  | name | class |
| ---- | ---- | ----- |
| 1    | 小谷 | 1班   |

<br />

## 1.脏写(Dirty Write)

事务A修改了另一个未提交事务B修改的数据。

| 时间编号 | Session A                           | Session B                           |
| -------- | ----------------------------------- | ----------------------------------- |
| 1        | begin;                              |                                     |
|          |                                     | begin;                              |
|          |                                     | Update set name='李四' where sno=1; |
|          | Update set name='张三' where sno=1; |                                     |
|          | commit;                             |                                     |
|          |                                     | Rollback;                           |

是没有任何隔离级别下，session A看到的效果是无效，明明提交了，却没有效果。

（按mysql默认隔离级别，session A的update语句会阻塞）

<br />

## 2.脏读(Dirty Read)

事务A读取到了事务B未提交的数据。

| 时间编号 | Session A                                        | Session B                           |
| -------- | ------------------------------------------------ | ----------------------------------- |
| 1        | begin;                                           |                                     |
|          |                                                  | begin;                              |
|          |                                                  | Update set name='李四' where sno=1; |
|          | select * from where sno=1;<br />(李四，脏读发生) |                                     |
|          | commit;                                          |                                     |
|          |                                                  | Rollback;                           |

是没有任何隔离级别下，session A读到了session B未提交的数据。

<br />

## 3.不可重复读(Non-Repeatable Read)

事务A读取了一个字段，然后事务B更新了该字段，之后事务A再次读同一个字段，值不同了，就发生了不可重复读。

| 时间编号 | Session A                                   | Session B                           |
| -------- | ------------------------------------------- | ----------------------------------- |
| 1        | begin;                                      |                                     |
|          | Select * from where sno=1;<br />（读出小谷) |                                     |
|          |                                             | Update set name='李四' where sno=1; |
|          | select * from where sno=1;<br />(读出李四)  |                                     |
|          | commit;                                     |                                     |

<br />

## 4.幻读(Phantom)

事务A读取了一些数据，然后事务B插入了一些新的行，之后事务A再次读取，多出几行，就发生了幻读。

| 时间编号 | Session A                                        | Session B              |
| -------- | ------------------------------------------------ | ---------------------- |
| 1        | begin;                                           |                        |
|          | Select * from where sno>0;<br />（读出小谷)      |                        |
|          |                                                  | Insert (2, 赵六, 2班); |
|          | select * from where sno>0;<br />(读出小谷、赵六) |                        |
|          | commit;                                          |                        |

<br />

更为准备地说是，第一次select得到的结果数据无法支撑后续的业务操作。比如说，select出来某记录不存在，准备插入此记录，但执行insert时却发现此记录存在，此时也算是幻读。

| 时间编号 | Session A                                           | Session B              |
| -------- | --------------------------------------------------- | ---------------------- |
| 1        | begin;                                              |                        |
|          | Select * from where sno>0;<br />（读出小谷id=1)     |                        |
|          |                                                     | Insert (2, 赵六, 2班); |
|          | Insert (2, 赵五, 2班);<br />(报主键错误)            |                        |
|          | Select * from where sno>0;<br />（只能读出小谷id=1) |                        |

mysql的RR是可以避免幻读的，通过select for update加锁来解决。

<br />

# SQL的四种隔离级别

read uncommitted：读到其他未提交事务的数据，不能避免脏读、不可重复读、幻读。

read committed：只能读到其他事务已提交的数据，解决了脏读，但仍有不可重复读、幻读。

repeatable read：解决脏读、不可重复读，但仍有幻读。

Serializable：串行。

四种级别都解决了脏写。

<br />

SQL标准下：

读未提交	----------	解决

读已提交	----------	脏读

可重复读	----------	不可重复读

串行化		----------	幻读

<br />

Mysql有点不一样：

读未提交	----------	

读已提交	----------	脏读

可重复读	----------	不可重复读、幻读 (采用MVCC、Next-Key Lock)

<br />

# MVCC

怎么理解MVCC？

采用乐观锁思想，用更好的方式处理读写冲突，用快照读做到非阻塞并发读。

<br />

## 隐藏字段、Undo Log版本链

Trx_id：最近更新记录的事务id。

Roll_pointer：每次修改记录时，会把旧的版本写入undo日志。指针就指向这条记录位置。

举例：假设插入记录的事务id为8，则该记录示意图如下：

```php
 id       name      trx_id  roll_pointer
 ---      -----     -----      
| 1 |    | 张三 |   | 8  |       指向 ---------------> insert undo
 ---      -----     -----
```

事务提交后，roll_pointer就没了，insert undo就被回收。

之后两个事务id10、20对这条记录进行update操作：

| 时间编号 | 事务10                         | 事务20                         |
| -------- | ------------------------------ | ------------------------------ |
| 1        | begin;                         |                                |
|          |                                | begin;                         |
|          | Update name='李四' where id=1; |                                |
|          | Update name='王五' where id=1; |                                |
|          | Commit;                        |                                |
|          |                                | Update name='钱七' where id=1; |
|          |                                | Update name='宋八' where id=1; |
|          |                                | commit;                        |

形成的版本链：

```php
 id       name      trx_id  roll_pointer
 ---      -----     -----      
| 1 |    | 宋八 |   | 20  |    指向----
 ---      -----     -----            |
                                     |
 ------------------------------------
 |
 |--> 1	钱七	20 -->
   		1 王五	10 -->
   		1	李四	10 -->
   		1 张三	8		.
```

<br />

## ReadView

每来一个事务，就生成一个ReadView，一对一。

ReadView包含的数据：

1. creator_trx_id：对应的事务id。
2. trx_ids：当前活跃的事务id列表（未提交的事务）。
3. up_limit_id：活跃事务中最小的事务Id。
4. low_limit_id：当前系统应该分配的下一个事务id。

<br />

ReadView取数据规则：

* 记录的trx_id == RV.creator_trx_id，可取。

* 记录的trx_id < RV.up_limit_id，可取。

* 记录的trx_id > RV.low_limit_id，不可取，取版本链下条记录判断。

* 记录的trx_id在up与low之间时：

  对于Read Committed，trx_id存在于trx_ids，说明活跃，不可取；不存在则可取。

  对于Repeatable Read，不可取。

<br />





