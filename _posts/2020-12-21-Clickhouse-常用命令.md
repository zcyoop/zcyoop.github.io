---
title:  'Clickhouse 常用命令'
date:  2020-12-21
tags: clickhouse
categories: [数据库,clickhouse]
---

## 数据表基本操作

```sql
-- 追加新字段
ALTER TABLE tb_name ADD COLUMN [IF NOT EXISTS] name [type] [default_expr] [AFTER name_after]
ALTER TABLE testtable ADD COLUMN colum1 String DEFAULT 'defaultvalue';

-- 修改字段类型
ALTER TABLE tb_name MODIFY COLUMN [IF EXISTS] name [type] [default_expr];
ALTER TABLE testtable MODIFY COLUMN age Int32;

-- 修改备注
ALTER TABLE tb_name COMMENT COLUMN [IF EXISTS] name 'some comment';
ALTER TABLE testtable COMMENT COLUMN key '主键ID';

-- 删除已有字段
ALTER TABLE tb_name DROP COLUMN [IF EXISTS] name;
ALTER TABLE tb_name DROP COLUMN key;

-- 修改数据表名称
RENAME TABLE default.testcol_v1 TO db_test.testcol_v2
```

## 分区操作

```sql
-- 查询分区信息
SELECT partition_id,name,table,database FROM system.parts WHERE table = 'partition_v2'

-- 删除指定分区
ALTER TABLE tb_name DROP PARTITION partition_expr
ALTER TABLE testtable DROP PARTITION 201907

```

## 查询数据库和表容量

```sql
-- 查看数据库容量
select
    sum(rows) as "总行数",
    formatReadableSize(sum(data_uncompressed_bytes)) as "原始大小",
    formatReadableSize(sum(data_compressed_bytes)) as "压缩大小",
    round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes) * 100, 0) "压缩率"
from system.parts;

-- 查询test表，2019年10月份的数据容量
select
    table as "表名",
    sum(rows) as "总行数",
    formatReadableSize(sum(data_uncompressed_bytes)) as "原始大小",
    formatReadableSize(sum(data_compressed_bytes)) as "压缩大小",
    round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes) * 100, 0) "压缩率"
from system.parts
	-- 根据实际情况加查询条件
    where table in('test')
        and partition like '2019-10-%'
    group by table;		
```

## 查看和删除任务

```sql
-- 这个命令和mysql是一样的
show processlist；
-- 如果进程太多，也可用通过查询系统表 processes，
select * from system.processes;
-- 指定主要关心字段
select 
  user,query_id,query,elapsed,memory_usage
from system.processes;

--  通过上面指令获取到进程相关信息后，可以用query_id条件kill进程
KILL QUERY WHERE query_id='2-857d-4a57-9ee0-327da5d60a90' 
-- 杀死default用户下的所有进程
KILL QUERY WHERE user='default'
```

*--  参考*

- [运维查看数据库及表容量](https://blog.csdn.net/yyoc97/article/details/103111464)
- Clickhouse原理解析与应用全实践分析