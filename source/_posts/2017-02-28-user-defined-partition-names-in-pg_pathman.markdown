---
layout: post
title: "User-defined partition names in `pg_pathman`"
date: 2017-03-01 11:06:00 +0300
comments: true
categories: 
---

One of the most frequiently asked questions about `pg_pathman` we get is how to specify a user-defined naming scheme for new partitions. Even though there is no standard way to do this, our suggestion is to write a callback function for partitioned table. Being set callback function performs some additional actions right after new partition is created.
A callback function takes JSONB value as an argument which contains some essential information about partition being created. This json may look something like this (for RANGE-partitioned table):

```sql
{
	"parent": "my_table",
	"parent_schema": "public",
	"parttype": "2",
	"partition": "my_table_1",
	"partition_schema": "public",
	"range_min": "1",
	"range_max": "101"
}

```

or (for HASH-partitioned table):

```sql
{
	"parent": "my_table",
	"parent_schema": "public",
	"parttype": "1",
	"partition": "my_table_0",
	"partition_schema": "public"
}
```
So let's say we have `my_table` table which we want to be partitioned by range with one month interval.

```sql
create table my_table (dt timestamp not null);
```

And we want our partitions to be named like `abc_2016_01`, `abc_2016_02`, etc. In this case callback function may look like this:

```sql
create or replace function my_callback(params jsonb)
returns void as
$$
declare
    range_min    timestamp;
    new_relname  text;
begin
    range_min := (params->>'range_min')::timestamp;

    -- generate new name for partition based on its parent name and its lower bound
    new_relname := format('%s_%s',
        params->>'parent',
        to_char(range_min, 'YYYY_MM'));

    -- rename partition
    execute format('alter table %s.%s rename to %s',
        params->>'partition_schema',
        params->>'partition',
        new_relname);
end
$$
language plpgsql;
```

Now let's assign the callback for the table, perform partitioning and see what will happen:

```sql
# select set_init_callback('my_table', 'my_callback(jsonb)');
# select create_range_partitions('my_table', 'dt', '2016-01-01'::timestamp, '1 month'::interval, 3);
 create_range_partitions 
-------------------------
                       3
(1 row)
# select * from pathman_partition_list;
  parent  |      partition      | parttype | partattr |      range_min      |      range_max      
----------+---------------------+----------+----------+---------------------+---------------------
 my_table | my_table_2016_01_01 |        2 | dt       | 2016-01-01 00:00:00 | 2016-02-01 00:00:00
 my_table | my_table_2016_02_01 |        2 | dt       | 2016-02-01 00:00:00 | 2016-03-01 00:00:00
 my_table | my_table_2016_03_01 |        2 | dt       | 2016-03-01 00:00:00 | 2016-04-01 00:00:00
(3 rows)
```
Great, all partitions have been renamed!

Now, renaming partitions isn't the only usage of callback functions. For example, you can setup the data rotation mechanism as described [here]({% post_url 2016-10-21-data-rotation-with-pg-pathman %}). Or automatically distribute partitions between few tablespaces. Let's see how it can be done:

```sql
-- suppose we have three tablespaces
create tablespace ts_0 location '/path/to/ts_0';
create tablespace ts_1 location '/path/to/ts_1';
create tablespace ts_2 location '/path/to/ts_2';

-- we will use this counter to determine tablespace number for new partition
-- based on round-robin algorithm
create sequence ts_counter;

create or replace function my_callback(params jsonb)
returns void as
$$
declare
    ts_name  text;
begin
	-- calculate tablespace name
    ts_name = 'ts_' || nextval('ts_counter') % 3;

    -- move partition to the tablespace
    execute format('alter table %s.%s set tablespace %s',
        params->>'partition_schema',
        params->>'partition',
        ts_name);
end
$$
language plpgsql;

-- create table and partition it
create table my_table (id serial);
select set_init_callback('my_table', 'my_callback(jsonb)');
select create_range_partitions('my_table', 'id', 1, 100, 7);
```

Let's see what we got:

```sql
select t.spcname, p.* from pathman_partition_list as p
join pg_class as c on c.oid = p.partition::regclass
join pg_tablespace as t on t.oid = c.reltablespace;

 spcname |  parent  | partition  | parttype | partattr | range_min | range_max 
---------+----------+------------+----------+----------+-----------+-----------
 ts_1    | my_table | my_table_1 |        2 | id       | 1         | 101
 ts_2    | my_table | my_table_2 |        2 | id       | 101       | 201
 ts_0    | my_table | my_table_3 |        2 | id       | 201       | 301
 ts_1    | my_table | my_table_4 |        2 | id       | 301       | 401
 ts_2    | my_table | my_table_5 |        2 | id       | 401       | 501
 ts_0    | my_table | my_table_6 |        2 | id       | 501       | 601
 ts_1    | my_table | my_table_7 |        2 | id       | 601       | 701
(7 rows)
```

Yep, partitions were distributed between tablespace as required. And all partitions that will be created in future will also be destributed the same way.

Another use case for callback functions is to keep the most recent data in a faster tablespace resided on SSD and move old data to a slower tablespace on HDD. The only pitfall here is that this operation can be pretty time consuming and will lock the transaction which triggers partition creation. So it would be better for callback function to add a task to a schedule and leave all the hard work to a scheduler.

Good luck!