# Import Data into the ODS Layer
## Problem Point
Before cleaning the data, we need to import the data into ods layer first.But how to import ?
## Solution
### Full Batch Mode
If the source database contains only a few tables, we can use full batch mode to import data. This is simple and easy.
I don't introduce the details.
### Incremental Mode
If the number of source database's table is large, we can use incremental mode to import data. This is more complex.
First,you should confirm the source table has incremental field,For example,the timestamp field. I will introduce two ways to implement incremental mode.
1. Use Overwrite Partition Mode
First,you should create ods table with partition which is the incremental field.You Should load the data fully at first,then you load partial data by to the middle table,then you can overwrite the ods table by the middle table,the pseudo-code is as follows:
```sql
//if target table is hive table
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
insert overwrite table ods_table partition (partition_field)
select a,b,c,partition_field
from mid_table
// For an Iceberg table, the following is an illustrative example of dynamic partition overwrite
SET spark.sql.sources.partitionOverwriteMode = dynamic;
insert overwrite table ods_table
select a,b,c,dt
from mid_table;
WHERE dt BETWEEN '2026-10-16' AND '2026-10-22';
```
2. Use Merge Into Mode
MERGE INTO is useful when the target table must support inserts and updates based on a business key.
This approach is commonly used with Iceberg tables when the SQL engine supports the required MERGE INTO operations.
For example, suppose both the source and target tables contain a unique id field.
```sql
merge into ods_table as target
using mid_table as source
on target.id = source.id
when matched then update set target.a = source.a,target.b = source.b,target.c = source.c
when not matched 
then insert (a,b,c,id) values (source.a,source.b,source.c,source.id)
```
### Real Time model
If the data is generated in real time, we can use real time model to import data. This is more complex.For example, we can use Flink CDC to import data in real time.
I will introduce the details in the other part.

Thank you for reading.You can leave your comments if you have any good ideas.
