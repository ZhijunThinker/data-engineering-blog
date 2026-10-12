# The most common problems in SQL JOIN Operations
## Problem point
When we join multiple tables to generate a new table, sometimes we find that the result is not what we expected.
The most common problems are as follows:
1. Duplicate rows or unexpected row explosion in the target table.
2. Missing rows in the target table.
## Reasons for the Problem
1. Duplicate rows or row explosion are usually caused by a Cartesian product when the right table contains duplicate rows in the join condition.
2. Missing matching rows are usually caused by NULL values in join keys.
## Solution
1. To avoid duplicate rows or row explosion in the target table, you should remove duplicates.
   1. The simplest method is to use GROUP BY on the join key,for example, pseudo-code:
```sql
    select
        xxx
    from l_table l 
    left join 
    (
        select 
            id,max(c) 
        from r_table 
        group by 
        id
    ) r
    on l.id = r.id
```
This method is not recommended because it loses some detail columns.
    2. Another method is to use ROW_NUMBER(). For example, pseudo-code:
```sql
     select
        xxx
    from l_table l 
    left join 
    (
        select 
            *
        from 
        (
        select 
            *,row_number() over(partition by id order by id) as cn 
        from r_table 
        ) as ll where cn = 1
    ) r
    on l.id = r.id
```
This method is my recommendation because it keeps all detail columns.
2. To avoid missing rows in the target table.You should check NULL values in the left table.However, sometimes it is inevitable.
you can use "NVL(left_table_column,'')" in the LEFT JOIN condition. For example, pseudo-code:
```sql
    select
        xxx
    from l_table l 
    left join  r_table  r
    on nvl(l.id,'') = nvl(r.id,'')
```

Thank you for reading.You can leave your comments if you have any good ideas.