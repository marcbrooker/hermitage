Testing Amazon Aurora DSQL transaction isolation levels
=======================================================

[Amazon Aurora DSQL](https://aws.amazon.com/rds/aurora/dsql/) is a distributed, scalable, SQL database. Aurora
DSQL offers a single isolation level (snapshot isolation), which we believe to be equivalent to PostgreSQL's
repeatable read level. Aurora DSQL using an optimistic concurrency control (OCC) approach, which means that
transactions never block, and so conflicts are visible as conflicting transactions aborting.

Setup (before every test case):

```sql
create table test (id int primary key, value int);
insert into test (id, value) values (1, 10), (2, 20);
```

Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

Aurora DSQL prevents Write Cycles (G0):

```sql
begin; -- T1
begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2
update test set value = 21 where id = 2; -- T1
commit; -- T1
select * from test; -- T1. Shows 1 => 11, 2 => 21
update test set value = 22 where id = 2; -- T2
commit; -- T2 (aborts, conflict with T1)
select * from test; -- either. Shows 1 => 12, 2 => 22
```

Aurora DSQL prevents Aborted Reads (G1a):

```sql
begin; -- T1
begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
abort;  -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

Aurora DSQL prevents Intermediate Reads (G1b):

```sql
begin; -- T1
begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11
commit; -- T2
```

Aurora DSQL prevents Circular Information Flow (G1c):

```sql
begin; -- T1
begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

Aurora DSQL prevents Observed Transaction Vanishes (OTV):

```sql
begin; -- T1
begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2
commit; -- T1
begin; -- T3
select * from test where id = 1; -- T3. Shows 1 => 11
update test set value = 18 where id = 2; -- T2
select * from test where id = 2; -- T3. Shows 2 => 19
commit; -- T2 (aborts, conflict with T1)
select * from test where id = 2; -- T3. Shows 2 => 19
select * from test where id = 1; -- T3. Shows 1 => 11
commit; -- T3
```

Predicate-Many-Preceders (PMP)
------------------------------

Aurora DSQL prevents Predicate-Many-Preceders (PMP):

```sql
begin; -- T1
begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

Aurora DSQL prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
begin; -- T1
begin; -- T2
update test set value = value + 10; -- T1
delete from test where value = 20;  -- T2
commit; -- T1
commit;  -- T2 (aborts, conflict with T1)
```


Lost Update (P4)
----------------

Aurora DSQL prevents Lost Update (P4):

```sql
begin; -- T1
begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2
commit; -- T1
commit; -- T2 (aborts, conflict with T1)
```

Read Skew (G-single)
--------------------

Aurora DSQL prevents Read Skew (G-single):

```sql
begin; -- T1
begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

Aurora DSQL prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
begin; -- T1
begin; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

Aurora DSQL prevents Read Skew (G-single) -- test using write predicate:

```sql
begin; -- T1
begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1
commit; -- T1 (aborts, conflict with T2)
```


Write Skew (G2-item)
--------------------

Aurora DSQL does not prevent Write Skew (G2-item):

```sql
begin; -- T1
begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```

Aurora DSQL does prevent Write Skew (G2-item) with use of `for update`:

```sql
begin; -- T1
begin; -- T2
select * from test where id = 2 for update; -- T1
select * from test where id = 1 for update; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2 (aborts, conflict with T1)
```

Anti-Dependency Cycles (G2)
---------------------------

Aurora DSQL does not prevent Anti-Dependency Cycles (G2):

```sql
begin; -- T1
begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

