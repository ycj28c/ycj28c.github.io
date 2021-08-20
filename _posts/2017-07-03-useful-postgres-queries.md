---
layout: post
title: Useful Postgres Queries
disqus: y
share: y
categories: [Database]
tags: [Postgres]
---

* convert table to json  

```sql
select to_json(pc) from proxy_company pc;
```

* converting a whole row to json; one row for each json  

```sql
SELECT
  pc.company_id,
  row_to_json(pc)
FROM proxy_company pc;
```

* aggregate the result and to json; building {company_id: [{row}, {row}]}
  
```sql
SELECT
  t.company_id,
  row_to_json(t)
FROM (
       SELECT
         pc.company_id,
         json_agg(row_to_json(pc)) fiscal_years
       FROM proxy_company pc
       GROUP BY company_id
     ) t order by company_id;
```

* check current user connection  

```sql
SELECT * FROM pg_stat_activity where state = 'active';
```

* check max connection setting  

```sql
show max_connections;
```

* find the lock the pid and kill it  

```sql
--for example, you know the 'market_index' table is frozen
select * from pg_locks where granted and relation = 'market_index'::regclass;

select * from pg_stat_activity where pid in (select distinct(pid) from pg_locks);

select * from pg_stat_activity where pid = '28769';

select pg_terminate_backend(28769);
```

*  pg_dump the mview and its index  

```shell
pg_dump -sOx -t cdna_search -h 10.1.50.35 -U insight insight_qa > cdna_search.sql
```

*  postgres foreign link related  

```sql
select * from pg_foreign_server;
select * from pg_user_mappings;
alter server aserver options (set host 'a.com', set dbname 'a_server');
alter user mapping for bserver server aserver options (set user 'usera', set password 'xxx');
```

*  rollback changes  

```sql
begin;
-- your query
rollback;
```

* how to query the array contains any  

```sql
select * from (
SELECT MCS.COMPANY_ID,ARRAY_AGG(DISTINCT S.SECTOR_DESCRIPTION)::text[] SECTORS
                    FROM MASTER_COMPANY_SECTOR MCS, SECTOR S
           WHERE MCS.SECTOR_ID=S.SECTOR_ID GROUP BY MCS.COMPANY_ID) msc
where msc.SECTORS && ARRAY['Technology', 'Telecom Technologies'];
```

*  recursive query  
[Find Parent Recursively using Query](https://stackoverflow.com/questions/3699395/find-parent-recursively-using-query)  

```sql
WITH RECURSIVE tree(child, root) AS (
   select c.executive_id, c.merged_to_executive_id from executive c join executive p on c.merged_to_executive_id = p.executive_id WHERE p.merged_to_executive_id IS NULL
   UNION
   select executive_id, root from tree
   inner join executive on tree.child = executive.merged_to_executive_id
)
SELECT * FROM tree where child = 135477;
```

*  compare two query data  

```sql
create temporary table tmp1 as select * for user where id = 1;
create temporary table tmp2 as select * for user where id = 2;

-- is data missing in tmp1
select * from tmp1
except
select * from tmp2;

-- is data missing in tmp2
select * from tmp2
except
select * from tmp1;
```

* could not read block 65802 in file "base/16387/180507": read only 0 of 8192 bytes issue fix  
[PostgreSQL: 末尾块收缩如pg_type pg_attribute异常和patch](https://yq.aliyun.com/articles/72687)  
[Error: Could not read Block X of relation base/Y/Z](https://dba.stackexchange.com/questions/44508/error-could-not-read-block-x-of-relation-base-y-z)  

```sql
--find wrong path
SELECT pg_filenode_relation(0, 180507);

--try reindex
REINDEX INDEX table1_index;

--vacuum db
vacuum analyze table1;
vacuum full verbose table1;  

--full db vaccum if don't know which broken
vacuum analyze
```

* how to use lateral  

```
#if source data look like below
#year         |     sh_out_dt      | sh_out          |
#-----------------------------------------------------
#2765265,0,34 | 2149524,287,4      | 1584011,5738,10 |

#we want to convert to below
#filed_name | document_id | field_offset | field_length |
#--------------------------------------------------------
#year       |  2765265    |       0      |      34      |
#sh_out_dt  |  2149524    |     287      |       4      |
#sh_out     |  1584011    |     5738     |      10      |

SELECT
  sub_query.name                                           AS field_name,
  NULLIF(split_part(sub_query.col, ',', 1), '') :: BIGINT  AS document_id,
  NULLIF(split_part(sub_query.col, ',', 2), '') :: NUMERIC AS field_offset,
  NULLIF(split_part(sub_query.col, ',', 3), '') :: NUMERIC AS field_length
  split_part(sub_query.col, ',', 1),
  REGEXP_SPLIT_TO_ARRAY(col, ','),
ARRAY_LENGTH(REGEXP_SPLIT_TO_ARRAY(col, ','), 1)
FROM data_ddown fyd
  LATERAL ( -- Getting column name and column data of corresponding column
  VALUES (TEXT 'year', fyd.year),
         (     'sh_out_dt', fyd.sh_out_dt),
         (     'sh_out', fyd.sh_out)
  ) sub_query(name, col)
WHERE col IS NOT NULL
  AND ARRAY_LENGTH(REGEXP_SPLIT_TO_ARRAY(col, ','), 1) >= 2;
```

*  performance analyze related  

```sql
--统计信息
select * from pg_stat_database;
--缓存命中率，如果低于1，可尝试调整shared_buffers
select blks_hit::float/(blks_read + blks_hit) as cache_hit_ratio from pg_stat_database where datname=current_database();
--事务提交率,低于1，检查是否死锁或其他超时太多
select xact_commit::float/(xact_commit +xact_rollback) as successful_xact_ratio from pg_stat_database where datname=current_database();
--优化后建议执行以下语句，方面对比优化前后数据
--pg_stat_reset()
--表级统计信息
select * from pg_stat_user_tables;
--索引使用率
select sum(idx_scan)/(sum(idx_scan) + sum(seq_scan)) as idx_scan_ratio from pg_stat_all_tables where schemaname='insight';
select relname,idx_scan::float/(idx_scan+seq_scan+1) as idx_scan_ratio from pg_stat_all_tables where schemaname='insight' order by idx_scan_ratio asc;
--开启
--shared_preload_libraries='pg_stat_statements'
--pg_stat_statements.track=all
create extension pg_stat_statements;
SELECT * FROM pg_available_extension_versions WHERE name = 'pg_stat_statements';
--语句级统计信息 通过pg_stat_statements ,postgres 日志、auto_explain 来获取
select * from pg_stat_statements;
--查询平均执行时间最长的3条查询
select calls,total_time/calls as avg_time,left(query,80) from pg_stat_statements order by 2 desc limit 3;
```

* convert the regclass in postgressql 
 
```sql
-- you may get some id from postgres log
-- for example: "process 9097 acquired AccessShareLock on relation 220216116 of database 16387 after 2741065.823 ms"
-- here is the query to translate
select * from pg_class where oid = '220216116'::regclass;
-- now we we it represent the "stock_price_on_or_after"
```

* find special character  

```sql
SELECT regexp_replace(bio, '([^[:ascii:]|’|“|”|–|…|™|è|—])', '[\1]', 'g') AS t_marked
FROM raw_data
WHERE bio ~ '[^[:ascii:]|’|“|”|–|…|™|è|—]' limit 20;
```

* create the sequence postgres way  

```sql
ALTER TABLE tableA ADD COLUMN IF NOT EXISTS tableA_id BIGSERIAL PRIMARY KEY;

--- Above single liner equivalent to below operations
CREATE SEQUENCE tableA_id_seq;
ALTER TABLE tableA ADD COLUMN IF NOT EXISTS tableA_id BIGINT PRIMARY KEY DEFAULT nextval('tableA_id_seq') NOT NULL;
ALTER SEQUENCE tableA OWNED BY tableA.tableA_id;
```

* find functions using certain functions  

```sql
with raw as (
  SELECT distinct(proname) || '\(' as dis_name
     FROM pg_proc
  WHERE prosrc ILIKE '%oracle%'
    and proname !~* 'substr'
)
select n.nspname, r.dis_name, proname, prosrc, proargnames
from pg_proc pp
join pg_namespace n on n.oid = pp.pronamespace
join raw r on pp.prosrc ~* r.dis_name
where nspname in ('schema_space');
```

* find mview using certain functions  

```sql
with raw as (
  SELECT distinct(proname) || '\(' as dis_name
     FROM pg_proc
  WHERE prosrc ILIKE '%oracle%'
    and proname !~* 'substr'
)
select schemaname, r.dis_name, matviewname, matviewowner, ispopulated, definition
from pg_matviews pm
join raw r on pm.definition ~* r.dis_name
where schemaname in ('xpfeed');
```

* find the table size

``` sql
select pg_size_pretty(pg_relation_size('table_name'));
```

* foreign table schema update  

``` sql
-- Every time changed the foreign db schema, need drop and reimport
DROP FOREIGN TABLE foreignDB.tableA;
import foreign schema foreignDB limit to (tableA) from server db_link into foreignDB;

-- drop all then import whole insight schema
select 'DROP FOREIGN TABLE if exists foreignDB.' || table_name || ';'  drop_ft_statement
from information_schema.tables
where table_schema = 'foreignDB' and table_type = 'FOREIGN TABLE';
import foreign schema foreignDB from server db_link into foreignDB;
```
