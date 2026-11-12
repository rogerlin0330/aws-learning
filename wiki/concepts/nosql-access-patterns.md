---
title: NoSQL Access Patterns and Key Design
type: concept
tags: [nosql, dynamodb, data-modeling, partition-key, sort-key, indexes, hot-partitions]
certs: [aws-saa]
updated: 2026-11-11
---

# NoSQL Access Patterns and Key Design

With [DynamoDB](../services/dynamodb.md), begin with **what the application needs to retrieve**, then choose keys and indexes that make those reads direct. A flexible item schema does not mean every search is equally efficient. The design connects data organization, throughput distribution, and [consistency requirements](db-transaction-model.md).

## Start With Questions the Application Must Answer

An access pattern states the known inputs, expected results, ordering, frequency, and acceptable staleness. For a weather application:

| Application question | Inputs and output | Candidate access path |
| --- | --- | --- |
| What was this station's observation at this time? | Station ID and timestamp; one item | Complete base-table key with `GetItem` |
| What did this station report yesterday? | Station ID and time range; ordered items | `Query` on station partition key and timestamp sort key |
| Which observations triggered alarms? | Alarm state; observations across stations | Sparse GSI keyed by alarm state |
| Which observations meet an unexpected research criterion? | Arbitrary attributes across history | Occasional scan or a separate analytics path |

Record approximate request rates and item sizes alongside this list. A dashboard refreshed thousands of times per second deserves a deliberately designed query path. An infrequent administrative search might reasonably use a scan. For broad analytical work, consider moving data into an [S3](../services/s3.md) dataset through a suitable [data-integration](data-integration.md) workflow, rather than repeatedly scanning the transactional table.

## Partition Keys Distribute Work; Sort Keys Group Related Reads

A partition key groups related items logically and helps locate them physically. A sort key organizes items within that group. For station history, `StationId` plus an ISO-formatted timestamp supports chronological range queries without a new table for every week.

The grouping must also distribute traffic. A table keyed only by country may have many items but only a few busy key values. A large country can dominate the workload. A key with many possible values helps only if **actual requests** spread across those values: one extremely popular item can still be a bottleneck.

[DynamoDB adaptive capacity](../services/dynamodb.md#regional-architecture-and-access) helps uneven traffic, but cannot remove every per-partition limit. On-demand billing changes how throughput is acquired; it does not make key design irrelevant. Apply the same scrutiny to index keys as to table keys. See [AWS partition-key guidance](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html).

## Indexes Add Read Paths and Write Work

An [LSI](../services/dynamodb.md#lsi-another-ordering-within-one-station) keeps the table's partition key and changes the sort key. A [GSI](../services/dynamodb.md#gsi-find-alarms-across-stations) can regroup items using another partition key. The new access path costs storage and maintenance work; choose projected attributes based on what readers actually need.

A **sparse index** contains only items that have its required key attributes. For example, storing `AlarmState = ALARM` only on alarm observations creates an index containing just that subset. Removing the attribute removes index membership after propagation. Storing `AlarmState = NORMAL` on every normal observation makes those observations index members too.

The simple alarm example concentrates all alarms under one key. If traffic becomes excessive, an application might distribute alarms across buckets such as `ALARM#00` through `ALARM#15`, then query and merge those buckets. That trades write distribution for read fan-out and more application logic. Choose it from measured workload needs, not as an automatic improvement.

## Read Less Data Before Filtering

These operations occur at different stages:

1. **Key condition:** determines the key range DynamoDB evaluates.
2. **Filter expression:** removes evaluated items from the response.
3. **Response projection:** removes attributes from the response.

Only the first directly narrows the evaluated key range. A filter that returns two items from a thousand evaluated items still pays to evaluate the thousand. A narrow **stored index projection**, however, changes the size of the index entries being read. That is different from asking for fewer response attributes. See [Query capacity behavior](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.Other.html).

## Keep Freshness and Atomicity Explicit

A single application can have different requirements for different reads:

- Station history may tolerate a short delay and use eventually consistent reads.
- Confirming a just-written observation may need a strongly consistent base-table read.
- Updating an observation only if its version is unchanged needs a conditional write.
- Changing several related records together may need a transaction.
- A global application must decide whether asynchronous regional replication is acceptable.

Do not substitute a GSI, [DAX cache](../services/dynamodb.md#dynamodb-accelerator-dax), or global replica without revisiting freshness. Each can introduce an independent delay. Likewise, [TTL](../services/dynamodb.md#time-to-live-ttl) cleanup does not enforce application validity: a session is expired when its timestamp says so, even if its database item still exists.

## Where Relational Design Still Fits

[RDS](../services/rds.md) and [Aurora](../services/aurora.md) fit workloads needing relational joins, database-enforced relationships, and flexible SQL queries. DynamoDB fits known key-based access patterns with managed partitioning and low operational overhead. Both can support transactions; the decision should reflect the data access requirements, not “SQL means ACID, NoSQL means no ACID.”

In a [serverless architecture](../patterns/serverless-architecture.md), DynamoDB provides persistent state for otherwise ephemeral [Lambda](../services/lambda.md) functions. The application still owns appropriate keys, authorization, retry behavior, and business invariants.

> [!tip] Exam Tip
> A new search pattern does not automatically require a table scan. Check whether an LSI or GSI can support it, whether its consistency is sufficient, and whether the index's lifecycle rules permit adding it now.

## References

- Adrian Cantrill, DynamoDB lectures on capacity, query/scan, and secondary indexes; synthesized examples extended with workload-design implications. [Course provider](https://learn.cantrill.io/).
- [AWS — Partition-key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)
- [AWS — Query behavior and consumed capacity](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.Other.html)
- [AWS — Global secondary indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
- [DynamoDB service guide](../services/dynamodb.md)
- [Database transaction models](db-transaction-model.md)
