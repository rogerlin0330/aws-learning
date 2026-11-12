---
title: Database Transaction Models — ACID, BASE & CAP
type: concept
tags: [database, acid, base, cap-theorem, consistency, availability, partition-tolerance, eventual-consistency, rds, dynamodb, nosql, relational, transactions]
certs: [aws-saa]
updated: 2026-11-11
---

# Database Transaction Models — ACID, BASE & CAP

**ACID** describes transaction guarantees. **BASE** describes a design approach that permits temporary inconsistency. **CAP** explains a constraint on distributed systems during network partitions. They are not mutually exclusive product categories: [DynamoDB](../services/dynamodb.md) supports eventually consistent reads and ACID transactions, while [RDS](../services/rds.md) can combine transactional writes with asynchronous read replicas.

## The CAP Theorem (the foundation)

| Term | Meaning in CAP |
| --- | --- |
| Consistency | Operations behave as if using one current copy; reads respect completed writes |
| Availability | Every request to a non-failing node eventually receives a successful response, even during a partition |
| Partition tolerance | The system must handle communication failures separating its nodes |

**During a network partition, a distributed system cannot guarantee both linearizable consistency and availability for every operation.** A node cut off from its peers cannot always determine whether it has the latest state. It can delay/reject an operation to preserve consistency, or continue serving operations that may conflict with the other side.

“Pick any two of three” is an incomplete shortcut. Network failures are a condition to design for, not a feature to disable. CAP does not say a healthy system must always sacrifice either consistency or availability, and does not directly determine whether a database supports ACID transactions. See the [Gilbert and Lynch proof](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer6.pdf).

## ACID

| Property | Meaning | Transfer example |
| --- | --- | --- |
| Atomicity | All transaction changes commit, or none do | Debit and credit succeed together |
| Consistency | A transaction preserves the application's declared invariants | Balance constraints remain valid |
| Isolation | Concurrent operations obey the database's selected isolation guarantees | Other transactions do not observe unsupported intermediate states |
| Durability | Committed results persist through failures covered by the service guarantee | A successful transfer is not just an in-memory change |

ACID's **consistency** means valid data states; CAP's **consistency** concerns the visibility and ordering of operations across replicas. Isolation levels also vary; “ACID” alone does not establish that every operation uses the strongest possible isolation level.

Transactions introduce coordination costs, especially across many items or locations. That is a tradeoff to measure, not a rule that transactional databases cannot scale. Both relational engines and suitably scoped NoSQL operations can provide transactions.

## BASE

| Term | Meaning |
| --- | --- |
| Basically available | Design favors serving requests despite some failures or replica disagreement |
| Soft state | Replica state may evolve without a new user operation as synchronization progresses |
| Eventually consistent | If updates stop and communication succeeds, replicas converge |

BASE is a broad design description rather than a precise universal API contract. Soft state does not mean the database abandons all integrity rules. Eventual consistency does not specify a fixed maximum delay and does not mean committed data is randomly discarded. Applications must understand the actual service's guarantees and tolerate the stale views those guarantees allow.

## Separate Questions to Ask

| Requirement | Mechanism to investigate |
| --- | --- |
| Does this read reflect successful prior writes? | Read consistency and replica scope |
| Must these related changes commit together? | Transaction atomicity and isolation |
| Can another writer change the value between my read and write? | Conditional writes, version checks, or a suitable transaction |
| Must another Region immediately observe a completed write? | Cross-Region replication/consistency mode |

A strongly consistent read does not lock the item against later writers. An atomic transaction in one Region does not automatically remain atomic when its changes replicate elsewhere. A [cache](../services/dynamodb.md#dynamodb-accelerator-dax) adds another freshness boundary regardless of the backing database's guarantees.

## AWS Mapping: DynamoDB

[DynamoDB reads](../services/dynamodb.md#read-consistency-and-transactions) default to eventual consistency. Tables and LSIs also support strongly consistent reads; GSIs do not. Its transactional APIs provide scoped ACID operations, while conditional writes protect an update against a changed value.

The [global-table modes](../services/dynamodb.md#global-tables) make the distinction especially clear:

- **MREC:** asynchronous replication; a transaction is atomic in its source Region, but another Region can temporarily observe only part of its changes.
- **MRSC:** supports strongly consistent reads across participating Regions, but does not support transaction APIs. Global read consistency is not synonymous with multi-item transaction support.

Choose [access patterns](nosql-access-patterns.md), indexes, and consistency together. For example, a new GSI might answer a query efficiently while failing its read-after-write freshness requirement.

> [!tip] Exam Tip
> Relational joins and constraints suggest [RDS](../services/rds.md) or [Aurora](../services/aurora.md). Key-value/document access suggests DynamoDB. If the question requires ACID changes in DynamoDB, evaluate Transactions; do not reject DynamoDB solely because it is NoSQL. Check the requested Region scope and feature compatibility.

## References

- Cantrill SAA-C03 study notes — *ACID vs. BASE — Database Transaction Models*; refined while ingesting the DynamoDB lectures.
- [Gilbert and Lynch — Brewer's conjecture and the feasibility of consistent, available, partition-tolerant web services](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer6.pdf)
- [AWS — DynamoDB transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [AWS — DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS — Global table consistency and transaction scope](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html)
- [RDS](../services/rds.md)
