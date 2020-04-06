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

### 意向锁

`意向锁` 英文原文 `Intention Lock` 是表级锁，获得S锁或者X锁之前，必须先获得意向锁

两个概念：`共享意向锁 IS`，`排他意向锁 IX`

讨论意向锁的作用，先要了解到，InnoDB有行级锁，表级锁，上文的S,X都是行级锁（注意，**行级锁锁定的是索引，并非数据行**）

意向锁用来锁定整个数据表，考虑以下的场景，T1获得了行级锁，这时，T2希望获得表级锁，T2需要判断是否已经有会话获得了标记锁，还要去检查每一行是否有行级锁，否则就和行级锁表级锁的规则冲突了。如果没有意向锁，那么要扫描所有行是否被锁，但有了意向锁，只要检查当前表是否存在意向锁就可以了，因为规定，获得行级锁之前，必须获得意向锁。

```mysql
select * from yourtable where id=1 for update;
/*这里获得行级X锁，但在获得行级锁之前，系统会自动申请意向锁IX*/
```

