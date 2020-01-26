存储引擎：
MyISAM：MYD+MYI
表级锁，表损坏修复，check table tablename，repair table tablename
支持的索引类型
数据压缩：myisampack -p -f myIsam.MYI 数据压缩后，只读。
限制：非事务 只读

InnoDB 