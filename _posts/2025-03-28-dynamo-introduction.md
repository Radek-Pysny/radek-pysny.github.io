---
title: 'DynamoDB introduction'
tags: [DynamoDB]
---

## DynamoDB introduction

 - [DynamoDB architecture](#dynamodb-architecture)
      - [Table classes](#table-classes)
      - [Capacity modes and pricing](#capacity-modes-and-pricing)
 - [Partitions via constant hashing](#partitions-via-constant-hashing)
 - [Data types](#data-types)
 - [Primary key](#primary-key)
      - [PK (simple primary key)](#pk-simple-primary-key)
      - [PK and SK (composite primary key)](#pk-and-sk-composite-primary-key)
 - [Secondary Indexes](#secondary-indexes)
      - [LSI (Local Secondary Index)](#lsi-local-secondary-index)
      - [GSI (Global Secondary Index)](#gsi-global-secondary-index)
      - [Comparison of LSI and GSI](#comparison-of-lsi-and-gsi)
 - [Transactions](#transactions)
 - [DynamoDB streams](#dynamodb-streams)
 - [DynamoDB Accelerator (aka DAX)](#dynamodb-accelerator-aka-dax)
 - [API](#api)
 - [Honorable mentions](#honorable-mentions)
      - [TTL](#ttl)
      - [NoSQL Workbench](#nosql-workbench)
      - [Notes on data modeling](#notes-on-data-modeling)


---

---

### DynamoDB architecture

It might be used very often as a persistence solution, but those are bad scenarios:
 - complex query patterns (DynamoDB is not optimized for complex joins and subqueries);
 - multi-table transactions (DynamoDB has limit on number of items/rows);
 - data model complexity (too many GSIs and LSIs).

Fault tolerance is handled by using write leader and asynchronous replication to e.g. two
different availability zones (of the same AWS region).

```text
                                       |-----------|
                                       |           |
                                 |---> | replica 1 |
                   |--------|    |     |  (AZ-A)   |
  |--------|       |        |    |     |-----------|
  | client | <---> | leader | ---|
  ---------|       |        |    |     |-----------|
                   |--------|    |     |           |
                                 |---> | replica 2 |
                                       |  (AZ-B)   |
                                       |-----------|
```

In CAP theorem, DynamoDB is by default running in AP mode (high availability + eventual
consistency). This means all writes go to the leader and some clients might have stale
data reads from replicas in different availability zones.

However, one might use per-request flag to request CP mode (strong read-after-write consistency)
that forces all reads just from the leader. This feature is limited to just base table and LSIs.


#### Table classes

Two primary table classes offered by DynamoDB are **Standard** and **Standard-IA** (meaning Standard 
Infrequent Access). The default table class is Standard. One should compare throughput and storage
costs ratio. If the storage exceeds approximately 42 % of table's costs (information valid in 
August 2024 as stated in [1]), then it make sense to switch the table class to Standard-IA to 
reduce costs.


#### Capacity modes and pricing

Each partition can handle about 1 000 WCUs, 3 000 RCUs, and 10 GiB of data per partition.
**WCU** stands for **Write Capacity Unit**, and it corresponds to a single write operation of up to
1 KiB of data. In analogy, **RCU** stands for **Read Capacity Unit**, but it might present either
strong consistent read operation on a single item of up to 4 KiB of data. If eventual consistent 
read operation is sufficient, that single item can be up to 8 KiB of data. So, reading a single
item of 9 KiB will be charge either 2 RCUs for EC read or 3 RCUs for SC read operation.
Be careful about `Scan` operation that will behave like reading all items without any cost
optimizations. The last important price-affecting feature I am aware of are transactions, effectively
doubling the price (WCU for 0.5 KiB data written, RCU for 2 KiB SC read data or 4 KiB EC read
data).

For more manual approach of handling is available **provisioned capacity mode**, where requirement for
each base table (or GSI) is configured by required WCUs and RCUs per second. Unused capacity from sliding
window of 300 seconds can be used in a **burst capacity** helping with spikes. Then operations will start
throttling. DynamoDB's **adaptive capacity** helps handle imbalanced traffic by automatic distributing
throughput across partitions, so heavily accessed partitions will receive more capacity based on a workload.
In provisioned capacity mode are tables metered and billed hourly (in exact instant). In provisioned mode,
utilization of auto-scaling can be used to dynamically scale up or down based on varying workload. 

WO eliminate the need for capacity planning and forecasting is designed **on-demand capacity mode**.
One can scale-down to zero in a true serverless manner, so one is billed really based on actual number
of read and write requests handled by the table. Units are renamed to **RRU** and **WRU** (changing
word Capacity for **Request**) but their meaning is more or less the same. On-demand capacity algorithm
always tries to keep peak capacity above the last peaks of request counts. 

For monitoring and tracking of both capacity modes are used AWS CloudWatch metrics called
`ConsumedReadCapacityUnits` and `ConsumedWriteCapacityUnits` metrics.

Both capacity modes isolates heavily accessed items (aka **hot keys**) even into separate partitions.

When talking about all the capacity stuff, one might also mention that a single item stored in DynamoDB
is limited to 400 KiB. This is a hard limit needed to fulfil the performance requirements. To be
precise, it is important to note that this capacity is affected also by names of all attributes stored
within that item. So using overly long attribute names also directly affects performance, capacity and
pricing.


---

### Partitions via constant hashing

DynamoDB stores data in **partitions** (similar to shards of other DBMS) being an allocation
of storage for a table backed by SSD and automatically replicated across multiple Availability
zones across an AWS Region. Partition management is handled entirely automatically. Additional
partitions for a table are allocated when:
 - admin increase provisioned throughput settings beyond what the existing partitions might handle;
 - or if an existing partition fills to capacity and more storage space is required.

DynamoDB scale through **consistent hashing** on partition key.
The basic idea is based on circular address range (e.g. 16k) and hash of partition key
is used to index a place on that circle. One might either hit there a number of partition
of continue clockwise until a partition number is found. And this is the partition where
to insert/lookup the given item.

When partition is added, just one (the following) partition is rehashed.
On removal of partition, all its items are placed to the following partition.

```text
                                  |----|
                                  | p1 |
                                  |----|
                              **************
                          **********************
                       ****************************
                    ***********            ***********
                   ********                    ********
                 ********                        ********
                *******                            *******
               *******                              *******
               ******                                ******
              ******                                  ******
       |----| ******                                  ****** |----| 
       | p4 | ******                                  ****** | p2 |
       |----| ******                                  ****** |----|
              ******                                  ******
               ******                                ******
               *******                              *******
                *******                            *******
                 ********                        ********
                   ********                    ********
                    ***********            ***********
                       ****************************
                          **********************
                              **************
                                  |----|
                                  | p3 |
                                  |----|
```

Or here is an idea of a bit extended algorithm that includes virtual nodes with random
distribution. Random distribution of virtual nodes is achieved via the use of multiple,
unique hash functions.

```text
|----|
| p1 |
|----|
*******4**
**********3***
*************2***
         *******1***
             *****4**
               *****2**
                 ****1**
                  ****3**
                   ****2*
                    ****1*
                    ****3* |----| 
                    ****** | p2 |
                    ****4* |----|
```


---

### Data types

Categorized as follows:
 - **Scalar types**: `number`, `string`, `binary`, `Boolean`, and `null`.
 - **Document types**: complex structures with nested attributes of types `list` (heterogeneous ordered collection)
   and `map` (similar to JSON object).
 - **Set types**: multiple scalar values of types `string set`, `number set`, and `binary set. All limited just by
   400 KiB item limit.


For collation and comparison are used bytes of underlying UTF-8 encoding.

Data type descriptors:
 - `S` = `string`
 - `N` = `number`
 - `B` = `binary`
 - `BOOL` = `Boolean`
 - `NULL` = `null`
 - `M` = `map`
 - `L` = `list`
 - `SS` = `string set`
 - `NS` = `number set`
 - `BS` = `binary set`


---

### Primary key

Primary consist of either:
 - simple consisting of only one attribute (partition key PK)
 - composite consisting of two attributes (partition key PK and sort key SK)

During definition is needed attribute names, data types and role of each attribute:
 - `HASH` for partition key
 - `RANGE` for sort key

Primary key attributes has to have data type defined as one of: `string`, `number`, or `binary`.

Constraints for `string` and `binary` attributes of key (not just a primary key of base table):
 - partition key length between 1 and 2048 bytes
 - sort key length between 1 and 1024 bytes

Other attributes or data types does not need to be defined (**schemaless**).

> Examples of composite primary key:
>  - Chat system: chatID as PK; monotonically increasing messageID as SK
>  - E-shop: userID as PK; monotonically increasing orderID as SK
>  - Social media: userID as PK; postID as SK


#### PK (simple primary key)

To store and retrieve item operations are based just on a partition key value used as an input
for internal hashing algorithm yielding the target partition. Note that the items are not stored
in any sorted order. Each item's location is determined by the hash value of its partition key.

Partition key value has to be unique across the DynamoDB table.


#### PK and SK (composite primary key)

Hash of partition key is handled the same was as in the previous section. However, DynamoDB tends
to keep items of the same partition key close together and in order given by the sort key value.
Set of items of the same primary key value is called an **item collection**, and it is optimized
for efficient retrieval of ranges of items. If the table does not have local secondary index,
item collection is split over as many partitions as needed (to store the data and serve read
and write throughput).

Combination of partition key and sort key tuple has to be unique across the DynamoDB table.
One might read multiple values in a single operation `Query` iff all the items has the same
partition key value. Optionally, one might request a range of sort key values to be retrieved.
Request for single item selects the target partition using hash function and then scans sorted
item collection to find the desired item.


---

### Secondary Indexes

Amazon DynamoDB provides fast access to items in a table by specifying primary key values. However, 
many applications might benefit from having one or more secondary (or alternate) keys available, 
to allow efficient access to data with attributes other than the primary key. To address this, 
you can create one or more secondary indexes on a table and issue `Query` or `Scan` requests against 
these indexes (instead of against the base table).

Both types on indices (together with optional base table's sort key) might be implemented by a B-Tree.

Each index has to define projection kind stating which columns are stored within that particular index:
 - `KEYS_ONLY` (only PK and SK of index + PK and SK of base table)
 - `INCLUDE` (with explicit list of included, with forced PK and SK of base table),
 - and `ALL` (attributes of base table being projected into index).

Secondary index does not need its PK and SK combination being unique. Uniqueness of items within secondary
indices is always guaranteed due to included base table's PK and SK (even when they are not included in
list of `INCLUDE` projection kind).

Index can be used only for reading operations via `Query` and `Scan` (so single-item read operations are
not supported at all).


#### LSI (Local Secondary Index)

Local secondary index was introduced as the first possible secondary index to be used by DynamoDB.
Its very basic purpose is to allow to access base table data ordered using completely another attribute.

- **Same PK and any SK** (Partition Key and Sort Key). "Local" means that everything stays in the same partition.
  The primary key has to be composite (same PK as its base table and any SK from base table's string/number/binary
  attributes).
- **Consumes capacity from the base table**.
- **Read-only** and automatically kept in sync.
- Up to 5 LSIs per base table.
- Can be created only along the table, so no possibility to be added or removed later. Removal scenario is either
  migrate data into new base table or remove/replace LSI SK attribute from the data.
- Base and LSIs writes always go to the same partition:
    - More likely to see a hot partition.
    - Large partitions cannot be split, so mind being caped by 10 GiB limit per collection/partition
      (see `ItemCollectionSizeLimitExceededException`), but also shared throughput of 1 000 WCU
      and 3 000 RCU.
- Base table item + LSIs limited to 400 KiB.
- SC and EC reads, so not loosing possibility of **Strong read-after-write Consistency**. Any write made
  to a base table is deemed successful after it is commited to both base table and all its LSIs.
- **Extended lookup** is supported, so non-projected attributes might be automatically fetched for expense
  of an extra read throughput.

> Example of chat system + sorting by a number of attachments:
>  - Base table: chatID as PK; messageID as SK; attributes userID, attachCount, and content.
>  - LSI: chatID as PK; attachCount as SK; any additional projected columns (if needed).


### GSI (Global Secondary Index)

Later introduced secondary index to DynamoDB. They are very versatile in comparison to LSI, so they fit
to many possible use cases like sparse indexes or write sharding.  

- **Any PK and optionally any SK** (Partition Key and Sort Key). "Global" meaning any PK is possible.
  The primary key can be simple (PK) or composite (PK and SK).
- Implemented like a **shadow table** or **materialized view**. Including its own capacity provisioning.
  Does not compete with base table needs. Propagation of changes from base table consumes write units,
  but from GSI's separate budget.
- **Read-only** with changes to base table being propagated to GSIs.
- Supports up to 20 GSIs (soft limit).
- Can be removed and added later. Any later added GSI will back-fill (where only write units are being charged
  and read units of base table are not affected at all).
- Not limiting item size, so all 400 KiB just to store an item.
- Item collections can split (no size not throughput limit).
- Only **Eventual Consistent (so-called EC) reads**. Propagation of writes from base table to GSIs occurs
  asynchronously.
- With GSI scans and queries, only attributes projected into GSI can be requested. No possibility to retrieve
  attributes from the base tale on the background.
- GSI backpressure to base table's partition leading to throttling of application requests.
- As PK and SK on GSI cannot be updated directly, one delete followed by write operation might be needed. 
  Higher provision might be needed for GSI than to base table.

> Example of chat system + extension with page of all messages of the current user:
>  - Base table: chatID as PK; messageID as SK; attributes userID, attachCount, and content
>  - GSI: userID as PK; messageID as SK; any additional projected columns (if needed)


### Comparison of LSI and GSI

| LSI                             | GSI                                    |
|---------------------------------|----------------------------------------|
| Created only with base table.   | Created/deleted any time.              |
| Shared RCU/WCU with base table. | WCU/RCU independent from the table.    |
| Item collection size <= 10 GiB. | No size limits.                        |
| Hard limit = 5 per base table.  | Soft limit = 20 per base table.        |
| Supports SC and EC reads.       | Supports only EC reads.                |
| Same PK as base table.          | Unconstrained primary key schema.      |
| Extended lookup is possible.    | Only projected attributes can be read. |


---

### Transactions

DynamoDB supports ACID transactions. There might be used transaction across tables, but they are 
hard-limited by the count of items affected by the transaction (e.g. 100 actions on unique items).
Actions of transaction can target items of different tables, but not in different AWS account or
region, while no two actions can target the same item. 

In comparison to RDBMS transactions, there is no way to explicitly control the transaction using
commands to commit or do a rollback. DynamoDB transaction is single request that either finishes
successfully or emits and exception to signal issue and rollback done.

All put, update, and delete requests to be executed in **all-or-nothing** way must be included in 
a single `TransactWriteItems` request. Similarly, one can use transaction made just from read operations 
using `TransactGetItems` request. Combination of reading and writing in a transaction is not supported, 
but write items transaction can contain `ConditionCheck` operations able to roll the whole transaction back.
E.g. money transfer banking transaction is possible, as each update action can state a `ConditionExpression`,
therefore check of balance before withdraw of amount is possible.

If any item involved in read or write transaction is part of another inflight transaction,
the newer transaction is immediately cancelled with `TransactionConflict` exception and application
logic has to decide what to do next with it.

Transactions are implemented based on two-phase commit protocol which involves "PREPARE" phase (validation
of action errors, condition checks, throttling exceptions, transaction conflicts, and other sanity checks)
and "COMMIT" phase (prepared actions are committed in parallel on all storage nodes). Due to two-phase
commit protocol are all actions within transaction charged for double price.


---

### DynamoDB streams

An optional feature captures data modifications events in DynamoDB table (as it might be 
enabled per table separately). Events appear in the stream in near-real time in the same 
order those events occurred in the form of so-called **stream records** with 24-hour 
retention. Those are kinds of stream records:
 - Stream record for create item event contains the complete item.
 - Update stream record captures before and after image of touched attribute(s).
 - Deletion stream record captures image of item before it was deleted.

One might use e.g. λ to create a trigger sending e.g. welcome emails to each new customer
with filled `EmailAddress` attribute via Amazon SES. DynamoDB streams also enables solution
of data replication within or across AWS regions.


---

### DynamoDB Accelerator (aka DAX)

It is built-in in-memory caching layer. It is used as automatic read write-through cache.
In most cases, it could be enough to use DAX when caching is needed.

Caching if simple queries might lower latency a bit, but this is not crucial as DynamoDB
performs fairly well here. However, caching results of aggregate queries in DAX might bring
a decent performance boost.


---

### API

DynamoDB has API divided into four topics in the Developer Guide [2], so I adopted the same idea
also in this post. The last subsection contains just a few examples of requests.

Longer results of queries are divided into pages cut by maximum size of 1 MiB (or less). To retrieve
the following page, one has to pass pagination token passed in the previous response.


#### Control pane

Used for management of DynamoDB tables:
 - `CreateTable` might also optionally create secondary indexes and enable stream for the table.
 - `DescribeTable` return information about table's primary key schema, throughput settings, and index information.
 - `ListTables` returns list of table names (strings).
 - `UpdateTable` creates or removes global secondary indexes, modifies stream settings, or throughput settings.
 - `DeleteTable` removes the table and all its depending objects from DynamoDB.


#### Data pane

**Data pane** lets you perform CRUD operations on a data of the given table.

DynamoDB supports so-called **PartiQL** — SQL compatible language — with `ExecuteStatement`
(to read multiple items or write/update single item) and `BatchExecuteStatement`
(to read/write/update multiple items in a single batch).

> `SELECT` statement of `ExecuteStatement` API can be used for retrieval of items from multiple item collections.
> This is a kind of bonus feature in comparison `Query` allowing retrieval of multiple items but with the same PK.

Classic CRUD API for data pane is described by the following paragraphs.

Creating data:
 - `PutItem` write a single item with fully specified primary key attributes. No other attributes are needed.
 - `BatchWriteItem` writes up to 25 items to a table.

Reading data:
 - `GetItem` retrieves a single item (entire or just a subset of attributes) from the table by the primary key.
 - `BatchGetItem` retrieves up to 100 items from one or multiple tables.
 - `Query` retrieves all items (entirely of just a subset of attributes) with a specific partition 
   key from the given table. Optionally, a sort key condition might be applied to limit the number
   of retrieved items. Sort order might be chosen (by default ascending). The query might be called
   either on table or index with composite primary key.
 - `Scan` retrieves all items in the specified table or index. You can retrieve entire items, 
   or just a subset of their attributes. Optionally, you can apply a filtering condition to return
   only the values that you are interested in and discard the rest.

Updating data:
 - `UpdateItem` modifies one or more attributes in an item. You must specify the primary key for
   the item that you want to modify. You can add new attributes and modify or remove existing
   attributes. You can also perform conditional updates, so that the update is only successful 
   when a user-defined condition is met. Optionally, you can implement an atomic counter, which
   increments or decrements a numeric attribute without interfering with other write requests.

Deleting data:
 - `DeleteItem` deletes a single item specified by the primary key from the table.
 - `BatchWriteItem` (yes this is not a typo) can be used to delete up to 25 items from one or
   more tables.


#### DynamoDB streams

 - `ListStreams` returns all streams or just those of a single table.
 - `DescribeStream` returns information about the stream, e.g. ARN and where to begin reading 
  the first few stream records
 - `GetShardIterator` returns a **shard iterator** (a data structure to be used for retrieval
   of record from the stream).
 - `GetRecords` retrieve one or more records using the given shard iterator.


#### Transactions

PartiQL:
 - `ExecuteTransaction` a batch operation allowing CRUD operations to multiple items both within
   or across tables with a guaranteed all-or-nothing result.

Classic API:
 - `TransactWriteItems` a batch operation allowing put/update/delete operations to multiple items 
   both within or across tables with a guaranteed all-or-nothing result.
 - `TransactGetItems` a batch operation allowing multiple get operations to retrieve multiple items
   from one or more tables.


#### Examples

Reading a single item via `GetItem` request with just a few projected attributes might be defined
by the following payload:

```json
{
    "TableName": "Music",
    "Key": {
        "Artist": "No One You Know",
        "SongTitle": "Call Me Today"
    },
    "ProjectionExpression": "AlbumTitle, Year, Price"
}
```

Example of `Query` request:

```json
{
    "TableName": "Music",
    "KeyConditionExpression": "Artist = :a and begins_with(SongTitle, :t)",
    "ExpressionAttributeValues": {
        ":a": "No One You Know",
        ":t": "Call"
    }
}
```

The first example of update using `UpdateItem` request:

```json
{
    "TableName": "Music",
    "Key": {
        "Artist":"No One You Know",
        "SongTitle":"Call Me Today"
    },
    "UpdateExpression": "SET RecordLabel = :label",
    "ConditionExpression": "Price >= :p",
    "ExpressionAttributeValues": {
        ":label": "Global Records",
        ":p": 2.00
    }
}
```

And yet another example of update via `UpdateItem` request that atomically increment a counter:

```json
{
    "TableName": "Music",
    "Key": {
        "Artist":"No One You Know",
        "SongTitle":"Call Me Today"
    },
    "UpdateExpression": "SET Plays = Plays + :incr",
    "ExpressionAttributeValues": {
        ":incr": 1
    },
    "ReturnValues": "UPDATED_NEW"
}
```


---

### Honorable mentions


#### TTL

TTL (aka **Time To Live**) can be enabled on particular DynamoDB table and then might be defined a specific
attribute to store TTL expiration timestamp stored in UNIX epoch time format at the second granularity.
TTL allows you to define a per-item indication on no longer needed items. DynamoDB automatically deletes
expired items within undefined time window (e.g. within a few days of their expiration time). Nice is that
this action does not consume write throughput.


#### NoSQL Workbench

NoSQL Workbench is a cross-platform client-side GUI application that can be used for development, testing,
and operations on DynamoDB. One can also use it to start DynamoDB locally in Docker for local development
purpose. It consists from three modules:
 - _Data modeling_,
 - _Data visualization_, 
 - and _Operation building_ (good for building and testing queries). 


#### Notes on data modeling

NoSQL storage requires completely different approach than relational DB systems. There is no space for
3NF (aka third normal form), BCNF, or even higher normal forms. Patterns appreciated by NoSQL approach
use more from denormalization and storing multiple copies of data. Also joining of data from different
tables is far away from usual NoSQL operations. The main reason is to keep data scalable and enable
high throughput required by real-world applications.

**Data that needs to be accessed together must be stored together** concept is based on keeping a copy
of data for each use case. E.g. for music player, name of song might be kept in many models (list of
the most played songs, song detail, song download counter, etc.).

**Single table** approach is based on using item collections (primary key consisting from PK and SK).
All the items in collection are stored on the same partition (therefore they are having the same value of PK;
e.g. internal ID of song) while prefix of SK is used to determine different models (e.g. prefix 
`a#` is used for song details, `d#` is used for details of one download, `x#` is used for download summary, etc.).

**Multiple tables** approach sometimes fits better. Imagine modeling a game, so player profile is key model, 
and there is integrated peer-to-peer chat. Those are distinct use cases and there is no scenario with 
simultaneous access to player profile and chat messages. Also, backup solutions for those two different models
might be completely different (chat holds much more data, those data are not that important, older chat messages
does not need to be backed up at all, etc.).

Different approaches for **handling of large items**:
 - keep data compressed (e.g. using Snappy's S2 extension; decompression is needed on each item
   retrieval and filtering of compressed data is not possible at all);
 - keep in DynamoDB just path for large object stores in S3 bucket;
 - uncompressed storage with vertical partitioning (store data as several items smaller than 400 KiB).

**Vertical partitioning** is a key concept introduced by single-table approach. Breaking down data of
large entity (e.g. all information about each application user) into multiple items within the same
item collection (using user ID as PK) with different prefixes of SK to be able to work with atomic parts
(fetching just that piece of information needed for the current use case). For storing of user profile
update history together with other user-related data, PK is user ID and SK can be using schema of:
`u#2023-05-01T07:10:13.432Z#D0Z84HK`, where `u#` is prefix to denote user profile update record,
followed by ISO 8601 format timestamp (bringing natural ordering into retrieved sequence of updates),
followed by "unique" request ID separated by yet another `#` (not losing records if multiple profile 
updates might be stored using exactly the same timestamp), and all other needed data stored in item's
attributes like `old_state`, `new_state`, `action`, and so on. If you need to retrieve all user profile
changes in year 2023 or just during May 2023, you can do it easily. Similar approach is in shopping cart
combined with buy-later list using SK against values `active#pID_123` and `inactive#pID_321` and user
ID as PK.

Another approach used by vertical partitioning is based on reducing number of required GSIs can be based
on common column names `GSI1PK` and `GSI1SK`. So one GSI might be re-used for use cases again utilizing
uniquely prefixed value of `GSI1SK`.

Ideas introduced by vertical partitioning can optimize performance and costs for data with infrequent
updates. Usually one pay more for initial write (as writing multiple smaller items instead of one large),
but reading and update costs can be already optimized.

**Sparse index** is a GSI where PK is attribute, that appears just for a small part of items of base table.
If we have database of maintained IoT devices, device log base table will contain item per each device
check done by operator. In case of any suspicious behavior, operator can escalate it to a service guy.
So introducing spare GSI with `escalatedTo` attribute as its PK, we might already list all escalated
logs without need of costly full table scan.

Sparse index might be also useful to optimize access to via low-cardinality data like Boolean flags.
Each attribute of type Boolean (like `active`) can be transformed into two or more columns (like
`active`, `inactive`, `complete`) while just one is present (e.g. with value of `y`) in the base table
and each GSI will contain just disjoint part of items.

**Keys-only index** is a GSI with `KEYS_ONLY` projection kind. Those GSIs are significantly smaller in size
that their base tables or GSIs with `ALL` projection kind. Lookup to that GSI usually requires additional
base table lookup. This setup has benefits especially for write-heavy applications.

**Write sharding** is idea of enabling higher write/update/delete throughput, e.g. for low-cardinality
attributes like gender, by adding suffixes to value (e.g. random value in range `_0` to `_99`). Then
this low-cardinality attribute can be still used as a PK for GSI, but one has to be aware of count of those
manually handled shards and work with it accordingly in application codebase. Also, date of birth in format
`YYYYMMDD` can be also pretty tricky example how to partition data. To shard this properly, one can e.g.
add suffix calculated by modulo number of designed shards applied on user ID.

**Skewed read replica** is based on using GSI (with `ALL` projection kind) to effectively enable twice
the read throughput by reading from both base table and GSI (as a read replica). Mind that data in GSI
can only read in eventual consistency mode as they are naturally skewed by "replication lag". Using LSI
as read replicate is not possible at all (as it is sharing read throughput with its base table).

---

##### References

[1] A. Dhingra and M. Mackay, Amazon DynamoDB - the definitive guide: Explore enterprise-ready, serverless NoSQL with 
predictable, scalable performance. 2024. ISBN 978-1-80324-689-5.

[2] Amazon Web Services, Inc., Amazon DynamoDB - Developer Guide [online]. Available on: 
[https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html).
