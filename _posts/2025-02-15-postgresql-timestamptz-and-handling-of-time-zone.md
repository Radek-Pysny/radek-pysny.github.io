---
title: 'PostgreSQL: TIMESTAMPTZ and handling of time zone'
tags: [PostgreSQL]
---

## PostgreSQL: TIMESTAMPTZ and handling of time zone

 - [Local timezone](#local-timezone)
 - [Examples](#examples)
     - [The difference on the output side](#the-difference-on-the-output-side) 
     - [The difference on the input side](#the-difference-on-the-input-side) 
 - [Summary from the official documentation](#summary-from-the-official-documentation)
 - [Best practice](#best-practice)
     - [Timezone approach](#timezone-approach) 
     - [No timezone approach](#no-timezone-approach) 

---

---

PostgreSQL supports two timestamp data types `timestamp without time zone` and `timestamp with time zone`.

| full type name                | short type name | storage size |
|-------------------------------|-----------------|--------------|
| `timestamp without time zone` | `timestamp`     | 8 bytes      |
| `timestamp with time zone`    | `timestamptz`   | 8 bytes      |


---


### Local timezone

Data type with timezone works with something we can call "local timezone". Its built-in default value is `GMT`.
Often, it is overridden by the value of `Timezone` parameter in `postgresql.conf`.
For more details see section
[19.11.2. Locale and Formatting](https://www.postgresql.org/docs/current/runtime-config-client.html#RUNTIME-CONFIG-CLIENT-FORMAT)
of PostgreSQL official documentation.

For us, it is important that one can override the database server settings.
I am not sure myself if it might be configured via connection string.
However, one can execute one of the following commands to change it for the current client session:

```postgresql
SET SESSION TIME ZONE 'Europe/Prague';
SET SESSION TIME ZONE 'UTC';
```

To check the current timezone, one might use the following query:

```postgresql
SELECT current_setting('TIMEZONE');
```

| current\_timezone |
|-------------------|
| UTC               |


---


### Examples

The following example consists of commands for `psql` commandline tool. 
For the test was used local Docker instance of PostgreSQL v10.3.

Let us create a simple `test` table with one `timestamptz` column and one `timestamp` column.

```postgresql
CREATE TABLE test (timestamptz_column timestamptz, timestamp_column timestamp);
```

That table is used for two examples presenting behaviour of both 


#### The difference on the output side

At first, set UTC as the timezone of your local session.

```postgresql
set session time zone 'UTC';
```

Insert result of `now()` function call into both columns and check the content.

```postgresql
INSERT INTO tz VALUES (now(), now());
SELECT * FROM tz;
```

| timestamptz\_column           | timestamp\_column          |
|-------------------------------|----------------------------|
| 2025-02-14 20:49:22.790807+00 | 2025-02-14 20:49:22.790807 |

One can see the difference between output of both data types.
The one with time zone has `+00` suffix. Why `+00`? Because we are in UTC session.

Now we clean up the table, so we can continue.

```postgresql
TRUNCATE tz;
```


#### The difference on the input side

At first, set Europe/Prague as the timezone of your local session.

```postgresql
set session time zone 'Europe/Prague';
```

Now, one can insert exactly the same values into both columns.
One of inserted value is `2025-02-14T10:30:0+10:30` to have an example with timezone (namely `+10:30`).
The second inserted value is `2025-02-14T20:00:00` as an example without timezone.

```postgresql
INSERT INTO tz VALUES ('2025-02-14T11:30:0+10:30', '2025-02-14T11:30:0+10:30');
INSERT INTO tz VALUES ('2025-02-14T20:00:00', '2025-02-14T20:00:00');
SELECT * FROM tz;
```

| timestamptz\_column    | timestamp\_column   |
|------------------------|---------------------|
| 2025-02-14 02:00:00+01 | 2025-02-14 11:30:00 |
| 2025-02-14 20:00:00+01 | 2025-02-14 20:00:00 |

As one can see, there are big differences. Let us summarize it in the table:

|                        | `timestamptz` column   | `timestamp` column     |
|------------------------|------------------------|------------------------|
| value with timezone    | normalized to UTC      | given timezone ignored |
| value without timezone | current timezone added | stored as it is        |

> Normalization of timezone means, that the timezone information is used for correct transformation into UTC.
> Mind that current session timezone is set to Europe/Prague meaning +01:00, so the value is in the DB stored
> as 2025-02-14 01:00:00+00, but value stored as timestamptz is transformed for presentation into the current
> session's timezone.

To see the true value stored in the DB for `timestamptz` columns, one shall use UTC timezone.

```postgresql
set session time zone 'UTC';
SELECT * FROM tz;
```

| timestamptz\_column    | timestamp\_column   |
|------------------------|---------------------|
| 2025-02-14 01:00:00+00 | 2025-02-14 11:30:00 |
| 2025-02-14 19:00:00+00 | 2025-02-14 20:00:00 |

One shall always tidy up after each testing session.

```postgresql
TRUNCATE tz;
```


---


### Summary from the official documentation

I will include here two snippets from PostgreSQL documentation:

> All timezone-aware dates and times are stored internally in UTC. They are converted to local time in the zone 
> specified by the TimeZone configuration parameter before being displayed to the client.

> PostgreSQL assumes your local time zone for any type containing only date or time.


---


### Best practice

If you work on the design of PostgreSQL database, take one of the following approaches. They seem to be equivalent.
However, the first one feels more natural to me.


#### Timezone approach

 1. Use `timestamptz` (aka `timestamp with timezone`) data type.
 2. Ensure that all data passed to database has a reasonable value of timezone include 
    (so `'2026-01-01 00:00:00+01:00'` for data input in Europe/Prague on instant of beginning the year 2026).
 3. Be aware of your current timezone (anyway you can check the current timezone yourself on each value selected 
    from any of `timestamptz` columns, e.g. `+01`).


#### No timezone approach

 1. Use `timestamp` (aka `timestamp without timezone`) data type.
 2. Ensure that timezone normalization is handled before it is inserted into the database.
