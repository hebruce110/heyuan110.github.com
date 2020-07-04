---

title:      "数据仓库Amazon Redshift"
date:       2018-08-09 14:06:44
author:     "bruce"
toc: true
tags:
    - redshift
    - datawarehouse
    - mysql
---


## 前言

Amazon Redshift 是一个快速、可扩展的数据仓库，可以简单、经济高效地分析数据仓库和数据湖中的所有数据。Redshift 通过在高性能磁盘上使用 Machine Learning、大规模并行查询执行和列式存储可提供比其他数据仓库快十倍的性能。


## 1.对系统表优化：vacuum

• VACUUM操作分两个阶段执行：（1）对未排序区域中的行进行排序；（2）如有必要（95%），会将表结尾处的新排序的行与现有行合并。因此，如果您按排序键顺序加载数据，则 VACUUM操作的速度会很快。
VACUUM类型
• VACUUM DELETE ONLY：仅删除 vacuum。回收磁盘空间，但对新行不排序。
• VACUUM SORT ONLY：仅排序 vacuum。对新行排序，但不回收磁盘空间。
• VACUUM FULL：完全 vacuum。默认的vacuum。相当于VACUUM DELETE ONLY+ VACUUM SORT ONLY，但比两者串行运行更加高效。
• VACUUM REINDEX：完全 vacuum 重建索引。对交错排序键的表有意义，它重新分析表的排序键列中的值分配，然后执行完全 VACUUM 操作。VACUUM REINDEX 花费的时间比 VACUUM FULL 长得多。

估算vacuum大概多长时间运行完毕：
`select * from SVV_VACUUM_PROGRESS;`

vacuum结束后查看效果：
`select table_name, (elapsed_time/1000000) as elapsed_time_s, sort_partitions, row_delta, sortedrow_delta, block_delta from SVV_VACUUM_SUMMARY order by xid;`

查看vacuum结束后磁盘的回收率：
`select * from SVL_VACUUM_PERCENTAGE order by xid;`

分析表格的那条语句，排列一下属性后，下面的语句会看得更加清晰<https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_SVV_TABLE_INFO.html>：
`select "table", size as size_MB, pct_used, tbl_rows, encoded, diststyle, sortkey_num, sortkey1, sortkey1_enc, unsorted, skew_sortkey1, skew_rows, max_varchar, stats_off from SVV_TABLE_INFO order by size_MB DESC;`

- .Amazon Redshift不会自动回收和重用在您删除行和更新行时释放的空间。
- 当您使用 COPY、INSERT 或 UPDATE 语句更新表时，新行将存储在磁盘上的单独的未排序区域中，然后根据查询的需求进行排序。如果大量行在磁盘上保持未排序状态，依赖已排序数据的操作（例如，范围受限的扫描或合并联接）的查询性能可能会下降。因此，有必要运行VACUUM命令。VACUUM命令做的工作包括：（1）回收空间：将从已删除行中恢复空间。（2）还原排序顺序，提高查询性能。因此，对于带排序键的表，VACUUM 命令可确保表中的新数据在磁盘上完全排序。

另外，ANALYZE命令会更新统计元数据，这使查询优化程序能够生成更准确的查询计划。因此当添加、删除或修改大量行时，您应运行 VACUUM 命令，然后运行 ANALYZE 命令。

## 2.统计系统相关数据

`select * from SVV_TABLE_INFO;`
<https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_SVV_TABLE_INFO.html>

## 3.查看压缩表合适的格式

`analyze compression tablename;`
<https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_ANALYZE_COMPRESSION.html>

## 4.查看错误原因

`select * from STL_LOAD_ERRORS ORDER BY starttime DESC;`

