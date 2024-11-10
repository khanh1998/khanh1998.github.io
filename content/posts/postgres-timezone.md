---
date: '2022-08-20T20:21:06+07:00'
draft: false
title: 'Postgres Timezone'
tags: ['database']
---
## Check timezone of the current session
```sql
khanh=# show timezone;
 TimeZone
----------
 UTC
(1 row)
```
## Change timezone of a session
```sql
khanh=# set timezone='asia/ho_chi_minh';
SET
khanh=# show timezone;
     TimeZone
------------------
 Asia/Ho_Chi_Minh
(1 row)
```
you can get timezone names here: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
## *timestamp* and *timestamptz*
`timestamptz` aka `timestamp with time zone`, the time when return to the client will be converted to the timezone has picked in the session.

`timestamp` aka `timestamp without time zone`.

Internally, the database store it as a 64-bit integer microsecond offset since **2000-01-01** in PostgreSQL (by default). Both types don’t store any info related to timezone.

## `now()`
The now function will return a `timestamp with time zone` value, the timezone will be the default timezone of database server.
```sql
khanh=# show timezone;
     TimeZone
------------------
 Asia/Ho_Chi_Minh
(1 row)

khanh=# select now();
              now
-------------------------------
 2022-08-20 13:28:49.782568+07
(1 row)
```
```sql
khanh=# set timezone='utc';
SET
khanh=# show timezone;
 TimeZone
----------
 UTC
(1 row)

khanh=# select now();
              now
-------------------------------
 2022-08-20 06:29:34.177309+00
(1 row)
```
if you want to remove the timezone, just cast the value to type timestamp:
```sql
khanh=# show timezone;
 TimeZone
----------
 UTC
(1 row)

khanh=# select now();
             now
------------------------------
 2022-08-20 07:09:13.94525+00
(1 row)

khanh=# select now()::timestamp;
            now
----------------------------
 2022-08-20 07:09:28.795878
(1 row)
```

## At time zone
At time zone is a operator to convert `timestamptz` to `timestamp` and vice versa.

https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT
## `timestamptz` to `timestamp`
For example, the timezone in the database server now is UTC, so `now()` will return a timestamp with the timezone UTC. If I want to convert the result of `now()` to timezone +7, here is what I do:
```sql
khanh=# show timezone;
 TimeZone
----------
 UTC
(1 row)

khanh=# select now();
              now
-------------------------------
 2022-08-20 07:22:21.113171+00
(1 row)

khanh=# select now() at time zone 'asia/ho_chi_minh';
          timezone
----------------------------
 2022-08-20 14:22:24.534935
(1 row)
```
the result i got is a timestamp without time zone.

*7h at time zone 0* = *14h at time zone 7*
## `timestamp` to `timestamptz`
```sql
khanh=# show timezone;
 TimeZone
----------
 UTC
(1 row)

khanh=# select timestamp '2001-02-16 20:38:40' at time zone 'asia/ho_chi_minh';
        timezone
------------------------
 2001-02-16 13:38:40+00
(1 row)
```
In the above example, it adds timezone +7 information to timestamp 2001-02-16 20:38:40, to create a timestamp with time zone.

The result is display in default time zone of session which is UTC.

*20h at time zone +7* = *13h at time zone 0*
## Use cases
For example, at 12h, the session timezone is UTC +7, I insert a record to database, if I just query the time normally, this is what you get: **2022-08-21T12:06:52.299305Z** which is incorrect. Because the letter Z stands for UTC 0.

So when you query, you can convert `timestamp` to `timestamp with time zone +7`, by using `at time zone`:
```sql
select
	id,
	title,
	artist,
	price,
	created_date at time zone 'asia/ho_chi_minh'
from
        album
```
This is what you got after converting the time zone: **2022-08-21T12:06:52+07:00**

## References
https://www.cockroachlabs.com/blog/time-data-types-postgresql
https://www.postgresqltutorial.com/postgresql-date-functions/postgresql-now
