---
date: '2022-08-21T11:52:42+07:00'
draft: false
title: 'Which operator is faster: like vs ='
tags: ['database']
---
## Explain
Explaining the query will help you to estimate how expensive your query is and which query plan will be used.
```sql
khanh=# explain select g from grades where g = 100;
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Index Only Scan using g_index on grades  (cost=0.42..97.55 rows=4636 width=4)
   Index Cond: (g = 100)
(2 rows)
```
So, this query will use *Index-Only-Scan*. The **cost** of the query is ranging from 0.42 to 97.55, 4636 is number of estimated return **rows**, **width** is size of return data in byte.
## Explain Analyze
Explain analyze will both estimate the complexity of your query and execute the query to get the actual result.
```sql
khanh=# explain analyze select g from grades where g = 100;
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using g_index on grades  (cost=0.42..97.55 rows=4636 width=4) (actual time=0.117..49.535 rows=5006 loops=1)
   Index Cond: (g = 100)
   Heap Fetches: 0
 Planning Time: 0.280 ms
 Execution Time: 95.587 ms
(5 rows)
```
The total time is sum of planning time and execution time: 0.280 + 95.587 ms. The actual number of return rows is 5006 instead of 4636 as estimation. The actual cost is also cheaper than the estimated cost.
## Prepare data
We need to prepare some data to compare the speed of `like` and `=`.
```sql
CREATE TABLE grades (id serial PRIMARY KEY,
                                       g int, name text);


CREATE INDEX g_index ON grades(g);


INSERT INTO grades (g, name)
SELECT random()*100,
       substring(md5(random()::text), 0, floor(random()*31)::int)
FROM generate_series(0, 1000000);

VACUUM (ANALYZE,
        VERBOSE,
        FULL);
```
We will create a table with only two columns, and insert 1 million random rows into that table.
## Comparing on non-index column
First, I want to test `=` operator on column *name*:
```sql
khanh=# explain analyze select * from grades where name = '00d4e4727e671';
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..12946.94 rows=6 width=23) (actual time=45.986..47.617 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on grades  (cost=0.00..11946.34 rows=2 width=23) (actual time=31.551..31.573 rows=0 loops=3)
         Filter: (name = '00d4e4727e671'::text)
         Rows Removed by Filter: 333333
 Planning Time: 0.112 ms
 Execution Time: 47.783 ms
(8 rows)
```
This query we use `like`:
```sql
khanh=# explain analyze select * from grades where name like '00d4e4727e671';
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..12946.94 rows=6 width=23) (actual time=45.363..47.072 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on grades  (cost=0.00..11946.34 rows=2 width=23) (actual time=29.736..29.748 rows=0 loops=3)
         Filter: (name ~~ '00d4e4727e671'::text)
         Rows Removed by Filter: 333333
 Planning Time: 0.174 ms
 Execution Time: 47.209 ms
(8 rows)
```
There are not much different between `=` and `like without ‘%’`. let’s add `%` to the query:
```sql
khanh=# explain analyze select * from grades where name like '%00d4e4727e671%';
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..12955.24 rows=89 width=23) (actual time=58.162..60.058 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on grades  (cost=0.00..11946.34 rows=37 width=23) (actual time=45.619..45.630 rows=0 loops=3)
         Filter: (name ~~ '%00d4e4727e671%'::text)
         Rows Removed by Filter: 333333
 Planning Time: 0.370 ms
 Execution Time: 60.164 ms
(8 rows)
```
The execution time is a bit longer than before but not too much. 60ms compare to 47ms.
## Comparing on index column
Let’s create a new index on name column:
```sql
khanh=# CREATE INDEX name_index ON grades(name);
CREATE INDEX
khanh=#
```
Query with `=` operator:
```sql
khanh=# explain analyze select * from grades where name = '00d4e4727e671';
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=4.47..28.01 rows=6 width=23) (actual time=0.101..0.178 rows=1 loops=1)
   Recheck Cond: (name = '00d4e4727e671'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on name_index  (cost=0.00..4.47 rows=6 width=0) (actual time=0.065..0.083 rows=1 loops=1)
         Index Cond: (name = '00d4e4727e671'::text)
 Planning Time: 0.986 ms
 Execution Time: 0.376 ms
(7 rows)
```
After adding a index on name column, the query time is improve quite a lot. 47ms without an index, 1ms with an index. Here you can see the database use **Bitmap index scan** on *name_index*, so it actually use the index that we just create to speed up the query.

Next we take a look on whether the index improve query time on like operator:
```sql
khanh=# explain analyze select * from grades where name like '%00d4e4727e671%';
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..12955.24 rows=89 width=23) (actual time=52.171..53.985 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on grades  (cost=0.00..11946.34 rows=37 width=23) (actual time=41.987..41.998 rows=0 loops=3)
         Filter: (name ~~ '%00d4e4727e671%'::text)
         Rows Removed by Filter: 333333
 Planning Time: 0.372 ms
 Execution Time: 54.188 ms
(8 rows)
```
The query time is roughly 54 ms, so it means that the index don’t improve query time if you use `like` operator, in other words, index don’t involve to `like` operator. The database is still need to perform a `sequence scan` to scan all rows in table to find the expected one.

That is understandable, because when we use the pattern `%search_string%`, the database have no choice than loop through the whole table and perform comparing one by one.

But, what about pattern `text%`, you only put `%` at the end of search string, from my understanding, the database can use index to compare in this case.
```sql
postgres=# explain analyze select * from grades where name like '00d4e4727e671%';
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..12953.24 rows=89 width=23) (actual time=101.556..107.356 rows=0 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on grades  (cost=0.00..11944.34 rows=37 width=23) (actual time=32.084..32.085 rows=0 loops=3)
         Filter: (name ~~ '00d4e4727e671%'::text)
         Rows Removed by Filter: 333334
 Planning Time: 2.530 ms
 Execution Time: 107.742 ms
(8 rows)
```
This is weird, the database still perform and expensive sequence scan.

After a while of searching on the internet, I found another option when creating index on text column, it is `text_pattern_ops`.

```sql
DROP INDEX IF EXISTS name_index;
CREATE INDEX name_index ON grades (name text_pattern_ops);
```
Let's see if the new index improve the query time:
```sql
postgres=# explain analyze select * from grades where name like '00d4e4727e671%';
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using name_index on grades  (cost=0.42..8.45 rows=89 width=23) (actual time=0.172..0.174 rows=0 loops=1)
   Index Cond: ((name ~>=~ '00d4e4727e671'::text) AND (name ~<~ '00d4e4727e672'::text))
   Filter: (name ~~ '00d4e4727e671%'::text)
 Planning Time: 4.601 ms
 Execution Time: 0.475 ms
(5 rows)
```
It does! The database uses index scan instead of sequence scan like before, query time is significantly reduced.\
I also did tests on patterns like `%search_string%` or `%search_string`, as expected, the `text_pattern_ops` index can't help.
To understand why `text_pattern_ops` helps the search with pattern `search_string%`, please read the links that are attached in the References section below.

## Conclusion
If the column doesn’t have an index, query times of `=` and `like` are quite similar.

If the column has an index, it reduces significant query time of `=` operator. whereas, the `like` operator can’t use index to improve query time.

But there is an index option that lets database using index while performing the `like` operator, it is `text_pattern_ops`. With this option on, you can use `like` operator with a pattern like `search_string%`.
## References
https://thoughtbot.com/blog/reading-an-explain-analyze-query-plan
https://dba.stackexchange.com/questions/53811/why-would-you-index-text-pattern-ops-on-a-text-column
https://dba.stackexchange.com/questions/10694/pattern-matching-with-like-similar-to-or-regular-expressions/10696#10696
https://dba.stackexchange.com/questions/240930/postgresql-difference-between-collations-c-and-c-utf-8
