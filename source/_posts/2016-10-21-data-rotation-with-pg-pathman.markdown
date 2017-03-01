---
layout: post
title: "Data rotation with `pg_pathman`"
date: 2016-10-21 18:19:36 +0300
comments: true
categories: 
---

Sometimes it is good to have only the most recent data in database and to remove outdated rows as new data comes (e.g., user activity logs or data from hardware sensors). This can be easily achieved using [`pg_pathman`](https://github.com/postgrespro/pg_pathman)'s features. Let's say we have a temperature sensor and we need to store readings from it for the last few days.

```sql
create table sensor_data (id serial, dt timestamp not null, value numeric);

insert into sensor_data(dt, value) select t, random()
from generate_series('2016-10-01', '2016-10-10 23:59:59', '1 second'::interval) as t;
```

First we divide whole dataset into several partitions by timestamp field so that each partition would contain a piece of data for a one day interval:

```sql
create extension pg_pathman;
select create_range_partitions('sensor_data', 'dt', '2016-10-01'::timestamp, '1 day'::interval);
```

As you may know `pg_pathman` automatically creates new partitions on `INSERT` when new data exceeds existing partitions. Now the interesting part. Since version 1.1 you can add custom callback to your partitioned table which would trigger every time new partition is created. So if we want to keep only recent data and remove old partitions we could use a callback like following:

```sql
create or replace function on_partition_created_callback(params jsonb)
returns void as
$$
declare
    relation    regclass;
begin
    for relation in (select partition from pathman_partition_list
                     where parent = 'sensor_data'::regclass
                     order by range_min::timestamp desc
                     offset 10)
    loop
        raise notice 'dropping partition ''%''', relation;
        execute format('drop table %s', relation);
    end loop;
end
$$
language plpgsql;

select set_init_callback('sensor_data', 'on_partition_created_callback');
```

Note that callback must meet certain requirements:

* it has a single `JSONB` parameter;
* it returns `VOID`.

Few comments on the code above. `pathman_partition_list` is a view that contains all partitions that are managed by `pg_pathman`. We sort it by `range_min` field so that the newest partitions come first, then skip first ten and drop the rest. The `set_init_callback()` function installs callback into `pg_pathman`'s config.

Now every time we insert data that exceeds the range covered by partitions a new partition is created and the oldest one is automatically removed:

```sql
select partition, range_min, range_max from pathman_partition_list;
   partition    |      range_min      |      range_max      
----------------+---------------------+---------------------
 sensor_data_1  | 2016-10-01 00:00:00 | 2016-10-02 00:00:00
 sensor_data_2  | 2016-10-02 00:00:00 | 2016-10-03 00:00:00
 sensor_data_3  | 2016-10-03 00:00:00 | 2016-10-04 00:00:00
 sensor_data_4  | 2016-10-04 00:00:00 | 2016-10-05 00:00:00
 sensor_data_5  | 2016-10-05 00:00:00 | 2016-10-06 00:00:00
 sensor_data_6  | 2016-10-06 00:00:00 | 2016-10-07 00:00:00
 sensor_data_7  | 2016-10-07 00:00:00 | 2016-10-08 00:00:00
 sensor_data_8  | 2016-10-08 00:00:00 | 2016-10-09 00:00:00
 sensor_data_9  | 2016-10-09 00:00:00 | 2016-10-10 00:00:00
 sensor_data_10 | 2016-10-10 00:00:00 | 2016-10-11 00:00:00
(10 rows)

insert into sensor_data(dt, value) values ('2016-10-11 15:05', 0.5);

select partition, range_min, range_max from pathman_partition_list;
   partition    |      range_min      |      range_max      
----------------+---------------------+---------------------
 sensor_data_2  | 2016-10-02 00:00:00 | 2016-10-03 00:00:00
 sensor_data_3  | 2016-10-03 00:00:00 | 2016-10-04 00:00:00
 sensor_data_4  | 2016-10-04 00:00:00 | 2016-10-05 00:00:00
 sensor_data_5  | 2016-10-05 00:00:00 | 2016-10-06 00:00:00
 sensor_data_6  | 2016-10-06 00:00:00 | 2016-10-07 00:00:00
 sensor_data_7  | 2016-10-07 00:00:00 | 2016-10-08 00:00:00
 sensor_data_8  | 2016-10-08 00:00:00 | 2016-10-09 00:00:00
 sensor_data_9  | 2016-10-09 00:00:00 | 2016-10-10 00:00:00
 sensor_data_10 | 2016-10-10 00:00:00 | 2016-10-11 00:00:00
 sensor_data_11 | 2016-10-11 00:00:00 | 2016-10-12 00:00:00
(10 rows)
```
