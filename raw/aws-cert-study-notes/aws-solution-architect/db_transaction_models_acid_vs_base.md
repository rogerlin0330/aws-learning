# ACID vs. BASE — Database Transaction Models

## Overview

**ACID** and **BASE** are two **database transaction models** — both acronyms. They define how transactions behave to and from a database, which in turn governs how the database system itself is architected. This is a theoretical-but-testable topic; expect a small number of exam questions keyed off the trigger terms.

---

## Foundation: The CAP Theorem

A computer-science theorem underpinning both models. **CAP** = **C**onsistency, **A**vailability, **P**artition tolerance.

|Term|Meaning|
|---|---|
|**Consistency**|Every read receives the most recent write — or gets an error.|
|**Availability**|Every request receives a non-error response — but _without_ a guarantee it contains the most recent write.|
|**Partition tolerance**|The system can be made of multiple network partitions and keeps operating even if messages between nodes are dropped or error out.|

**The core rule:** Any database product can deliver a **maximum of two** of these three factors.

**Why?** Picture a multi-node database on a network. If communication fails between nodes (a partition), and someone reads from the database, you have two choices:

- **Cancel** the operation → lower availability, preserve consistency.
- **Proceed** with the operation → improve availability, risk consistency.

So when a database has multiple nodes and a network is involved, you generally must choose between **consistency** and **availability**. ACID and BASE pick opposite trade-offs:

> 🎯 **ACID focuses on consistency. BASE focuses on availability.**

---

## ACID

> 🎯 **Exam default:** If you see **ACID**, assume it refers to an **RDS** database. ACID-based design **limits scalability** due to its rigid rules.

|Letter|Property|Meaning|
|---|---|---|
|**A**|**Atomic**|Either _all_ parts of a transaction succeed, or _none_ do.|
|**C**|**Consistent**|Transactions move the database from one valid state to another — nothing in between.|
|**I**|**Isolated**|Concurrent transactions don't interfere; parallel execution yields the same result as if run sequentially.|
|**D**|**Durable**|Once committed, a transaction stays committed even through system/power failure or restart.|

**Atomic — bank transfer example:** Transferring $10 from account A to account B has two parts: (1) remove $10 from A, (2) add $10 to B. You never want one part to succeed while the other fails. Both apply, or neither applies.

**Consistent:** In relational databases there may be links between tables (an item in one table must have a corresponding item in another) or value-range rules. Every transaction must respect these and leave the DB in a valid state.

**Isolated:** Essential for running many transactions at once (from different apps/users). Each executes in full as though it were the only transaction running.

**Durable:** Once the database tells the application a transaction is complete and committed, that data is stored such that a failure, power loss, or node restart won't lose it.

**Why financial institutions use ACID:** Most relational platforms use ACID transactions because they implement a rigid, reliable form of data management — but that rigidity is exactly what limits scalability.

---

## BASE

> 🎯 **Exam default:** If you see **BASE**, assume a **NoSQL**-style database. BASE databases are **highly scalable** and **high performance** because they offload consistency concerns to the application.

|Letters|Property|Meaning|
|---|---|---|
|**BA**|**Basically Available**|Read/write operations are available as much as possible, but with **no consistency guarantee**. Data availability is achieved by spreading and replicating across all nodes.|
|**S**|**Soft state**|The database does **not** enforce its own consistency — it delegates that responsibility to the **developer/application**.|
|**E**|**Eventually consistent**|Consistency may happen _eventually_; wait long enough and a read will match a prior write — but not immediately by default.|

**Basically available:** Rather than enforcing immediate consistency, BASE-modeled NoSQL databases ensure availability by replicating data across nodes. Reads and writes are "kinda / maybe" — it does its best, but there's no guarantee.

**Soft state:** Breaks from the idea of a database that enforces its own consistency. The application must be aware of consistency and state. If you need immediate consistency (a read that always sees all prior writes), the application must **explicitly request it** — and only if the database supports it. Otherwise the app must tolerate reading data that differs from what another instance just wrote.

**Eventually consistent:** By default, reads under a BASE model are eventually consistent. Applications must tolerate the possibility that a read doesn't reflect the latest writes. Many databases can do _both_ eventually and immediately consistent reads — but the application has to be aware and **explicitly ask** for consistent reads.

---

## AWS Mapping: DynamoDB

DynamoDB normally works in a **BASE-like** way:

- Offers **both** eventually consistent and immediately consistent reads — your application must be aware of which it's requesting.
- Also offers **DynamoDB Transactions**, which provide **ACID** functionality.

---

## 🎯 Exam Quick Reference

|If you see...|Default assumption|
|---|---|
|**BASE**|NoSQL-style database|
|**ACID**|RDS database|
|**NoSQL / DynamoDB** _together with_ **ACID**|DynamoDB Transactions|

|Model|Optimizes for|Scalability|Typical use|
|---|---|---|---|
|**ACID**|Consistency|Lower (rigid rules)|RDS, relational, financial|
|**BASE**|Availability|High / high performance|NoSQL, DynamoDB|

---

## ⚠️ Traps & Gotchas

- ⚠️ **CAP = pick two of three**, but in practice, once a network/partition is involved, the real choice is **Consistency vs. Availability**. ACID → C, BASE → A.
- ⚠️ **ACID's "C" (Consistent) ≠ CAP's "Consistency."** ACID consistency means "moves between valid states per DB rules"; CAP consistency means "every read sees the latest write." The exam may blur these — read carefully.
- ⚠️ **BASE ≠ "bad."** It sounds weak ("kinda available," "soft state"), but it's deliberately trading guarantees for **scalability and performance**.
- ⚠️ **Eventual consistency is the BASE default**, not the only option — DynamoDB can do strongly/immediately consistent reads _if the app asks_.
- ⚠️ **DynamoDB can be ACID too** via DynamoDB Transactions — don't reflexively map DynamoDB to BASE if the question mentions transactions/ACID.

---

📚 _The lesson notes that ACID/BASE and CAP are deep topics with extra reading linked in the course; the above covers the exam-essential subset._