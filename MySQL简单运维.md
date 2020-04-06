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

