- [Online DDL](#online-ddl)
- [索引操作(Index Operations)](#索引操作index-operations)
  - [创建&添加索引](#创建添加索引)
  - [删除索引](#删除索引)
  - [添加全文索引](#添加全文索引)
  - [改变索引类型](#改变索引类型)
- [主键操作](#主键操作)
  - [添加主键](#添加主键)
  - [删除主键无添加](#删除主键无添加)
  - [删除主键并添加](#删除主键并添加)
- [列操作](#列操作)
  - [添加列](#添加列)
  - [删除列](#删除列)
  - [重命名](#重命名)
  - [重排序](#重排序)
  - [改类型](#改类型)
  - [设定默认值](#设定默认值)
  - [取消默认值](#取消默认值)
  - [改变自增长值](#改变自增长值)
  - [设定非空NOT NULL](#设定非空not-null)
  - [修改enum、set值](#修改enumset值)











#### Online DDL

mysql版本: 5.6

在线数据定义操作有：索引操作、主键操作、列操作、外键操作、表操作、分区操作

https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html#online-ddl-column-operations



#### 索引操作(Index Operations)

表览， *星号代表视情况而定.

| Operation                            | In Place | Rebuilds Table | Permits Concurrent DML | Only Modifies Metadata |
| :----------------------------------- | :------- | :------------- | :--------------------- | :--------------------- |
| Creating or adding a secondary index | Yes      | No             | Yes                    | No                     |
| Dropping an index                    | Yes      | No             | Yes                    | Yes                    |
| Adding a `FULLTEXT` index            | Yes*     | No*            | No                     | No                     |
| Changing the index type              | Yes      | No             | Yes                    | Yes                    |

##### 创建&添加索引

```mysql
CREATE INDEX name ON table (col_list);
ALTER TABLE tbl_name ADD INDEX name (col_list);
```

 执行期间不阻塞读写。



##### 删除索引

```mysql
DROP INDEX name ON table;
ALTER TABLE tbl_name DROP INDEX name;
```

 执行期间不阻塞读写。



##### 添加全文索引

```mysql
CREATE FULLTEXT INDEX name ON table(column);
```

首次添加全文索引，并且没有`FTS_DOC_ID`列（啥玩意儿?），会重建数据表，并阻塞读写，之后再添加的话就不会重建表, 但还是会阻塞读写。



##### 改变索引类型

```mysql
ALTER TABLE tbl_name DROP INDEX i1, ADD INDEX i1(key_part,...) USING BTREE, ALGORITHM=INPLACE;
```

 执行期间不阻塞读写。



#### 主键操作

| Operation                                 | In Place | Rebuilds Table | Permits Concurrent DML | Only Modifies Metadata |
| :---------------------------------------- | :------- | :------------- | :--------------------- | :--------------------- |
| Adding a primary key                      | Yes*     | Yes*           | Yes                    | No                     |
| Dropping a primary key                    | No       | Yes            | No                     | No                     |
| Dropping a primary key and adding another | Yes      | Yes            | Yes                    | No                     |

##### 添加主键

```mysql
ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;
```

*  执行期间不阻塞读写。
* 会重建表，拷贝数据，是个耗时的操作。

* 最好在create table的时候指定主键，如果在create table的时候没有指定主键，mysql为自动选择第一个UNIQUE && NOT NULL列为作主键，如果没有，msyql生成隐藏主键列。

* 重建过程：

  1. 生成一个带有主键的临时表A 

  2. 拷贝原表数据到临时表A

  3. 原表重命名为另一个临时表B

  4. 临时表A重命名为原表

  5. 删除临时表B

* 主键列不允许有重复值，且不能有NULL值(视ALGORITHM值而定)。

* `ALGORITHM=COPY` mysql会把NULL转成相应的字段类型默认值，比如说0(整型列)、空字符串(字符型列)等。这是一个非标准行为，官方建议不要用。
* `ALGORITHM=INPLACE` 只允许在[`SQL_MODE`](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_sql_mode) 包含`strict_trans_tables` 或 `strict_all_tables`标志，且列的值没有NULL时使用。更符合标准，官方推荐使用。

* `ALGORITHM=COPY`与`ALGORITHM=INPLACE`都需要拷贝数据，但官方建议使用`ALGORITHM=INPLACE`，因为性能更好，因为INPLACE：

  1. 没有`undo`、`redo`日志，这些日志操作都是比较费性能的。

  2. 二级索引是排好序的，可以按序装载（没懂啥意思）。

  3. 没用到change buffer缓冲，因为对二级索引没有随机插入。

     

##### 删除主键无添加

```mysql
ALTER TABLE tbl_name DROP PRIMARY KEY, ALGORITHM=COPY;
```

只有`ALGORITHM=COPY`支持删除一个主键，而不用再创建另一个新的主键。



##### 删除主键并添加

```mysql
ALTER TABLE tbl_name DROP PRIMARY KEY, ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;

```

跟添加主键操作一样，耗时。



#### 列操作

| Operation                                             | In Place | Rebuilds Table | Permits Concurrent DML | Only Modifies Metadata |
| :---------------------------------------------------- | :------- | :------------- | :--------------------- | :--------------------- |
| Adding a column                                       | Yes      | Yes            | Yes*                   | No                     |
| Dropping a column                                     | Yes      | Yes            | Yes                    | No                     |
| Renaming a column                                     | Yes      | No             | Yes*                   | Yes                    |
| Reordering columns                                    | Yes      | Yes            | Yes                    | No                     |
| Setting a column default value                        | Yes      | No             | Yes                    | Yes                    |
| Changing the column data type                         | No       | Yes            | No                     | No                     |
| Dropping the column default value                     | Yes      | No             | Yes                    | Yes                    |
| Changing the auto-increment value                     | Yes      | No             | Yes                    | No*                    |
| Making a column `NULL`                                | Yes      | Yes*           | Yes                    | No                     |
| Making a column `NOT NULL`                            | Yes*     | Yes*           | Yes                    | No                     |
| Modifying the definition of an `ENUM` or `SET` column | Yes      | No             | Yes                    | Yes                    |

##### 添加列

```mysql
ALTER TABLE tbl_name ADD COLUMN column_name column_definition, ALGORITHM=INPLACE, LOCK=NONE;

```

* 会重建表, 是个耗时操作

* 如果添加的是一个自增长的列，那执行期间不允许读写; 其他列可以

* 最少需要`ALGORITHM=INPLACE, LOCK=SHARED`(没懂)

  

##### 删除列

```mysql
ALTER TABLE tbl_name DROP COLUMN column_name, ALGORITHM=INPLACE, LOCK=NONE;
```

会重建表, 是个耗时操作



##### 重命名

```mysql
ALTER TABLE tbl CHANGE old_col_name new_col_name data_type, ALGORITHM=INPLACE, LOCK=NONE;
```

* 保留类型不变，执行期间允许读写；否则不能读写。

* 外键的说明略。

  

##### 重排序

```mysql
ALTER TABLE tbl_name MODIFY COLUMN col_name column_definition FIRST, ALGORITHM=INPLACE, LOCK=NONE;
```

会重建表, 是个耗时操作



##### 改类型

```mysql
ALTER TABLE tbl_name CHANGE c1 c1 BIGINT, ALGORITHM=COPY;
```

会重建表，且执行期间不允许读写。只支持`ALGORITHM=COPY`。



##### 设定默认值

```mysql
ALTER TABLE tbl_name ALTER COLUMN col SET DEFAULT literal, ALGORITHM=INPLACE, LOCK=NONE;
```

只会修改元数据，即.frm文件。不会重建表，不阻塞读写。



##### 取消默认值

```mysql
ALTER TABLE tbl ALTER COLUMN col DROP DEFAULT, ALGORITHM=INPLACE, LOCK=NONE;
```

与设定默认值一样，不会重建表，不阻塞读写。



##### 改变自增长值

```mysql
ALTER TABLE table AUTO_INCREMENT=next_value, ALGORITHM=INPLACE, LOCK=NONE;
```

只修改在内存中的值，而不是文件中的。

（我是这样理解的，自增长的值在内存中保留，每次来一条数据就+1，再写入文件。每次mysql重启会读表中的最大自增长值放在内存。）

不会重建表，允许读写。



##### 设定非空NOT NULL

```mysql
ALTER TABLE tbl_name MODIFY COLUMN column_name data_type NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
```

* 会重建表。
* [`SQL_MODE`](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_sql_mode) 包含`STRICT_ALL_TABLES` 或`STRICT_TRANS_TABLES`时才会生效。
* 列值包含NULL值，执行失败。
* 外键执行失败。



##### 修改enum、set值

```mysql
CREATE TABLE t1 (c1 ENUM('a', 'b', 'c'));
ALTER TABLE t1 MODIFY COLUMN c1 ENUM('a', 'b', 'c', 'd'), ALGORITHM=INPLACE, LOCK=NONE;
```

* 在ENUM、SET后面添加数据，且类型容量够的话，不重建表。
* SET最多存8个元素，占用1个字节，添加多一个元素，会扩展到2两字节，会重建表。

