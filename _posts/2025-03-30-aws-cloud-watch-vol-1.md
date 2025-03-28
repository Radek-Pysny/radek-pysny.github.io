---
title: 'AWS CloudWatch vol. 1'
tags: [AWS]
---

## AWS CloudWatch vol. 1

 - [AWS CloudWatch Log groups - filtering unstructured logs](#aws-cloudwatch-log-groups---filtering-unstructured-logs)
 - [AWS CloudWatch Log Insights](#aws-cloudwatch-log-insights)
 - [AWS CloudWatch Dashboard](#aws-cloudwatch-dashboard)

---

---

### AWS CloudWatch Log groups - filtering unstructured logs

If you face unstructured log events in AWS CloudWatch Log groups, then searching might be a bit tricky at
beginning. To make it more enjoyable, let us take together on some basic usage examples.

Filtering input is **case-sensitive**, so `ERROR` is different to `Error` or to `error`.
One does not need to enter full word, filtering just its part is enough.

| `ERROR` matches                    | `ERROR` does not match        |
|------------------------------------|-------------------------------|
| `[ERROR 400] BAD REQUEST`          | `error=cannot process ticket` |
| `[ERROR 401] UNAUTHORIZED REQUEST` | `rpc error`                   |
| `[ERROR 419] MISSING ARGUMENTS`    | `Error: entity not found!`    |
| `[ERROR 420] INVALID ARGUMENTS`    | `NotFoundError`               |
| `ERRORNEOUS PROGRAM`               | `ErRoR`                       |

Each space-separated word is taken as one term, so **multi-term filter** if formed by multiple words
(two or more). Terms are AND-ed (meaning all terms has to be matched to match the whole log).

| `ARGUMENT ERROR` matches        | `ARGUMENT ERROR` does not match    |
|---------------------------------|------------------------------------|
| `[ERROR 419] MISSING ARGUMENTS` | `[ERROR 400] BAD REQUEST`          |
| `[ERROR 420] INVALID ARGUMENTS` | `[ERROR 401] UNAUTHORIZED REQUEST` |

**Exact phrase match** can be used for either multi-word term (aka term including spaces),
but also term that contains some special characters (e.g. `[`, `]`, `)`, etc.).
Exact phrase match is always enclosed in double quotes (`"`).

| `"ERROR][grpc"` matches           | `"ERROR][grpc"` does not match |
|-----------------------------------|--------------------------------|
| `2025-03-28 [ERROR][grpc] failed` | `[ERROR] [grpc]`               |

Using question mark prefix (`?`) is denoted so-called **optional term**.
Using multiple optional terms can be used to express OR-ed terms.
Mind that combination of optional terms and "mandatory terms" will ignore oll optional terms
and only those without `?`-prefix will be used for filtering of logs.

| `?BAD ?INVALID` matches            | `?BAD ?INVALID` does not match     |
|------------------------------------|------------------------------------|
| `[ERROR 400] BAD REQUEST`          | `[ERROR 401] UNAUTHORIZED REQUEST` |
| `[ERROR 420] INVALID ARGUMENTS`    | `[ERROR 419] MISSING ARGUMENTS`    |

| `?"clean up" ?xyz` matches | `?"clean up" ?xyz` does not match |
|----------------------------|-----------------------------------|
| `map clean up`             | `abc xy yz xz`                    |
| `xyz`                      | `map is cleaned up`               |
| `xyz will do clean up`     | `[ERROR 419] MISSING ARGUMENTS`   |

So far, all introduced terms were **include terms**, so logs matching that term should be included.
However, one might also use so-called **exclude terms** by adding minus-prefix (`-`) to each of them.

| `ERROR -ARGUMENTS` matches         | `ERROR -ARGUMENTS` does not match |
|------------------------------------|-----------------------------------|
| `[ERROR 400] BAD REQUEST`          | `[ERROR 420] INVALID ARGUMENTS`   |
| `[ERROR 401] UNAUTHORIZED REQUEST` | `[ERROR 419] MISSING ARGUMENTS`   |


---

### AWS CloudWatch Log Insights

Mind that in UI of Logs Insights can be any output exported into clipboard.
My favourite format is Markdown.


#### Example 1

Let us have a log line similar to the following one:

```text
2025-03-30T21:29:42+0000 [INFO] operation finished (processing delay 1m6.055140945s) entity_id=cid-123
```

As one might see, log message holds two important fields: parameter "entity_id" and processing delay
baked into log message using formatted log (yes this is example of bad practice). Logs Insights allow
us to extract data using `parse` directive. Here is one of possible solutions: 

```text
filter @message ~= "operation finished"
| parse @message /\(processing delay (?<processing_delay>.+)\)/
| parse @message /entity_id=(?<client_id>\d+)\//
| sort @timestamp asc
| fields @timestamp
| limit 10000
```

That query can be translated into human-readable language using just a few bullet points:
 - filter logs with message containing term `operation finished`,
 - parse from `(processing delay 1m6.055140945s)` the duration value into **processing_delay** variable,
 - parse from `entity_id=cid-123` the ID value (e.g. "cid-123") into **client_id** variable,
 - sort logs by log timestamp in ascending order (the oldest goes first),
 - include **timestamp** variable in output (next to parsed ones),
 - and limit output to maximum of 10 000 lines.


#### Example 2

Imagine the following log lines:

```text
2025-03-28T21:07:53+0000 [INFO] perf-tracing: redis_dao.generateNickname time_us: 160 
2025-03-28T21:07:53+0000 [INFO] perf-tracing: grpc.processEntity time_us: 31521 entity_id=42 entity_type_id=12 prio=2
```

To extract enough information out of those logs, one might use the following Log Insights query:

```text
fields @timestamp, @message
| filter @message like 'perf-tracing:'
| parse @message "* [*] perf-tracing: * time_us: * *" as @logTime, @logLevel, @function, @time_us, @rest
| stats count(*), min(@time_us) as min, average(@time_us) as average, pct(@time_us, 90) as perc_90, pct(@time_us, 95) as perc_95, pct(@time_us, 99) as perc_99, max(@time_us) as max
by bin(1m), @function
```

That query does the following steps:
 - include fields **timestamp** and **message** (aka whole log message),
 - filter those containing exact term `perf-tracing:`,
 - parse from message 5 parts (each part is represented by `*`) into variables **logTime**, **logLevel**, etc.,
 - calculate count of logs and statistical values of variable **time_us** (namely minimum, average 90-percentile,
   95-percentile, 99-percentile, and maximum values),
 - calculate statistics from 1-minute bins and bins by value of **function** variable. 


---

### AWS CloudWatch Dashboard

To manage dashboards, one should be aware that each dashboard is described by JSON. If it has textual
representation, it can be stored in Git repository as other source code. In Actions is one item, that
presents the source code of dashboard. Source code can be copied, pasted, but also modified directly
in that simple editor. Each widget (chart, title, etc.) is represented by one item in "widgets"
array.

Dashboard supports variables. Those might be useful e.g. to prepare a radio button for selection of 
the currently presented environment. Idea based on text pattern replacement is described by the
following code snippet:

```text
{
   "variables": [
      {
         "type": "pattern",
         "pattern": "service-env",
         "inputType": "radio",
         "id": "service_env",
         "label": "environment",
         "defaultValue": "service-production",
         "visible": true,
         "values": [
            {
               "value": "service-test",
               "label": "\"test\""
            },
            {
               "value": "service-production",
               "label": "\"production\""
            }
         ]
      }
   ],
   "widgets": []
}
```

Those are two points, that were not so intuitive to me when I started to build my first dashboards.
