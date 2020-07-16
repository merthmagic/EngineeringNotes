# MySQL简单运维

## 查询信息

### 获取存储引擎信息

```mysql
show engines;
```

`mariadb` 10.3.0为例，可以获取到的结果为：

| Engine             | Support | Comment                                                      | Transactions | XA   | Savepoints |
| ------------------ | ------- | ------------------------------------------------------------ | ------------ | ---- | ---------- |
| CSV                | YES     | CSV storage engine                                           | NO           | NO   | NO         |
| Memory             | YES     | Hash based, stored in memory, useful for temporary tables    | NO           | NO   | NO         |
| MRG_MyISAM         | YES     | Collection of identical MyISAM tables                        | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                        | NO           | NO   | NO         |
| Aria               | YES     | Crash-safe tables with MyISAM heritage                       | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, foreign keys and encryption for tables | YES          | YES  | YES        |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                           | NO           | NO   | NO         |
| SEQUENCE           | YES     | Generated tables filled with sequential values               | YES          | NO   | YES        |



### 参数查询

```mysql
/*显示所有参数*/
SHOW VARIABLES;
/*定制化查询,例如查询事务相关参数*/
SHOW VARIABLES LIKE 'tx%';
/*查看字符集*/
SHOW VARIABLES LIKE 'character%';
/*查看排序规则*/
SHOW VARIABLES LIKE 'collation_%';
/*查看时区*/
SHOW VARIABLES LIKE "%time_zone%";
/*查看当前默认存储引擎*/
SHOW VARIABLES like "default_storage_engine";
```

### 查询当前连接 信息

```mysql
/*显示所有连接*/
SHOW PROCESSLIST;
/*定制化查询*/
SELECT * FROM information_schema.PROCESSLIST WHERE DB='XXX' AND HOST LIKE 'XXX%'
```

### 查询当前`InnoDB`事务

```mysql
SELECT * FROM information_schema.INNODB_TRX it;
```

### 查询表大小

```mysql
SELECT 
    table_name AS `Table`, 
    round(((data_length + index_length) / 1024 / 1024), 2) `Size in MB` 
FROM information_schema.TABLES 
WHERE table_schema = "your_db_name"
order by `Size in MB` desc
```

### 根据索引名称查询索引

```mysql
select * from TABLE_CONSTRAINTS where CONSTRAINT_NAME like 'UK_ji9v%'
```



## 锁和事务

### 常见锁

这里讨论的上下文基于`InnoDB`.

常见的锁包括`共享锁(S锁)`和`排他锁(X锁)`，共享锁是读锁，排他锁是写锁。

举例：假设两个事务T1、T2

T1获得S锁后，T2可以获得S锁，但不能获得X锁；

T1获得X锁后，T2无法获得S锁或者X锁。

获得锁的方式：

```mysql
/*S Lock*/
select * from yourtable where id=1 lock in share mode;
/*X Lock*/
select * from yourtable where id=1 for update;
```

#### 意向锁

`意向锁` 英文原文 `Intention Lock` 是表级锁，获得S锁或者X锁之前，必须先获得意向锁

两个概念：`共享意向锁 IS`，`排他意向锁 IX`

讨论意向锁的作用，先要了解到，InnoDB有行级锁，表级锁，上文的S,X都是行级锁（注意，**行级锁锁定的是索引，并非数据行**）

意向锁用来锁定整个数据表，考虑以下的场景，T1获得了行级锁，这时，T2希望获得表级锁，T2需要判断是否已经有会话获得了标记锁，还要去检查每一行是否有行级锁，否则就和行级锁表级锁的规则冲突了。如果没有意向锁，那么要扫描所有行是否被锁，但有了意向锁，只要检查当前表是否存在意向锁就可以了，因为规定，获得行级锁之前，必须获得意向锁。

```mysql
select * from yourtable where id=1 for update;
/*这里获得行级X锁，但在获得行级锁之前，系统会自动申请意向锁IX*/
```

TIP：意向锁是表级别的，用来防止有行级锁的情况下还去获得表级锁，行级锁申请意向锁不冲突，如果冲突，那一个表就只有一个行级锁了，这和行级锁的定义不符合

#### 行级锁

行级锁的实现有三种，`Record Locks`,`Gap Locks ` `Next-Key Locks`, 其中 Next-Key Locks是Record Locks和Gap Locks的结合.

> A record lock is a lock on an index record. For example, `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` prevents any other transaction from inserting, updating, or deleting rows where the value of `t.c1` is `10`.
>
> A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record. For example, `SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;` prevents other transactions from inserting a value of `15` into column `t.c1`, whether or not there was already any such value in the column, because the gaps between all existing values in the range are locked.
>
> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

详细需要参考 https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html

**什么时候会加锁**

> A [locking read](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_locking_read), an [`UPDATE`](https://dev.mysql.com/doc/refman/5.6/en/update.html), or a [`DELETE`](https://dev.mysql.com/doc/refman/5.6/en/delete.html) generally set record locks on every index record that is scanned in the processing of the SQL statement. It does not matter whether there are `WHERE` conditions in the statement that would exclude the row. `InnoDB` does not remember the exact `WHERE` condition, but only knows which index ranges were scanned. The locks are normally [next-key locks](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_next_key_lock) that also block inserts into the “gap” immediately before the record. However, [gap locking](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_gap_lock) can be disabled explicitly, which causes next-key locking not to be used. 

### 事务

#### 幻读

> [Origin Link](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/phantom-read.html#)
>
> **Phantom read**
>
> A Phantom read occurs when one user is repeating a read operation on the same records, but has new records in the results set:
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)READ UNCOMMITTED
>
> Also called a Dirty read. When this isolation level is used, a transaction can read uncommitted data that later may be rolled back. A transaction that uses this isolation level can only fetch data but can't update, delete, or insert data.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)READ COMMITTED
>
> With this isolation level, Dirty reads are not possible, but if the same row is read repeatedly during the same transaction, its contents may be changed or the entire row may be deleted by other transactions.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)REPEATABLE READ
>
> This isolation level guarantees that a transaction can read the same row many times and it will remain intact. However, if a query with the same search criteria (the same WHERE clause) is executed more than once, each execution may return different set of rows. This may happen because other transactions are allowed to insert new rows that satisfy the search criteria or update some rows in such way that they now satisfy the search criteria.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)SERIALIZABLE
>
> This isolation level guarantees that none of the above happens. In addition, it guarantees that transactions that use this level will be completely isolated from other transactions.
>
> Based on this information, we can provide basic guidelines for choosing the proper isolation level for the ODBC connection that is going to be used by Crystal Reports:
>
> READ UNCOMMITTED should be used with reports that do not rely on data accuracy. Usually these same reports also process/access a high number of records. This optimizes performance while executing your report with a minimum number of database locks. Examples of reports in this category:
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Statistical information at the end of a month.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Sales report covering a previous year.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Reports running daily that accesses data which is rarely updated.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Reports running daily if the values displayed are only used as indicators.
>
> COMMITTED READ should be used with reports running daily on data that is frequently modified. This enables good performance while executing reports on a "live" database with an average number of record locks that are immediately released. Examples of reports in this category include:
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Daily reports on regularly updated data requiring 100 percent accuracy at the time the report is processed.
>
> ![*](https://documentation.progress.com/output/ua/OpenEdge_latest/rpcry/bullet.png)Reports that provide snapshots of an operation at any time during the day. Such reports are used for monitoring purposes, such as stock exchange status reports.
>
> REPEATABLE READ and SERIALIZABLE should not be used with reports as they do not add value at the time the report is generated, especially when compared to COMMITTED READ.
>
> Having reviewed the transaction isolation levels, you can now configure your ODBC driver.

