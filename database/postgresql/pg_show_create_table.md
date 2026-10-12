# Record a Method to Show the CREATE TABLE Statement of a PostgreSQL Table
## Problem point
As a Data Engineer,ETL tasks are part of my daily work.Synchronizing a table from Postgresql to big data system 
is a common task.At first,we need to understand the structure of the source table.
Usually,we can show table structure easily like mysql database,However,sometimes,we can't do this,
Especially on some Cloud Platforms.For example,in Tencent's WeData and AWS，we can only 
operate the Postgresql Database through Sql interface provided by web page,rather 
than connecting to the database using tools such as Dbeaver and Navicat,let alone 
connecting via the commend line.
## Reasons for the Problem
For data security reasons, we can only operate the database through the SQL window provided by the webpage.
The sql statement "show create table db.table_name" is invalid in the pg,because pg datasase does not save the original
sql like "Create table sql" as mysql database do.
For example,you would do:
```sql
CREATE TABLE test(
 id int,
 name varchar(20)
);
```
Pg internally stores the following information:
1. Database objects
2. Field definition
3. Field type
4. Field constraint
5. index
Stored in the system catalogs:
```sql
pg_class
pg_attribute
pg_constraint
pg_indexes
```
## Solution
1. Use "pg_dump" to dump the table structure.but this can't operate in web sql window.
```bash
pg_dump \
-h 10.10.10.10 \
-U postgres \
-d db \
-t schema.table_name \
--schema-only
```
2. Use "pg_get_tabledef" function to get the table structure.
```sql
SELECT pg_get_tabledef('schema.table_name'::regclass);
```
However,execute in the native PostgreSQL:
```text
ERROR: function pg_get_tabledef(regclass) does not exist
```
this method only works in the Database developed based on the secondary utilization of the PostgreSQL database.
3. Use "select" statement to get the table structure.This method is the most flexible,
Also,this is my favorite method.
```sql
select 
c.attname as column_name,
format_type(a.atttypid, a.atttypmod) as column_type,
col_description(a.attrelid, a.attnum) as column_comment,
obj_description(c.oid) as table_comment
from pg_class c
join pg_attribute a on c.oid = a.attrelid
join pg_namespace n on c.relnamespace = n.oid
where 
c.relname = 'table_name'
and n.nspname = 'schema_name'
and a.attnum > 0
```

Thank you for reading.You can leave you comment if you have any good ideas.