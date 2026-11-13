---
title: Amazon Neptune
type: service
tags: [neptune, graph-database, property-graph, rdf, gremlin, opencypher, sparql, nodes, relationships, serverless, ncu, global-database, neptune-analytics, graphrag, fraud-detection, recommendations, knowledge-graph]
certs: [aws-saa]
updated: 2026-11-13
---

# Amazon Neptune

Amazon Neptune is a managed **graph database** service. In a graph database the **relationships** between items are as important as the items themselves, and they are stored and queried as first-class data. Neptune is built for data whose relationships are fluid and change often, and for questions that are mainly about how things are connected.

For the SAA exam, Neptune is a **selection** topic, not a deep dive. Know what it is, that its architecture resembles [Aurora](aurora.md), and the scenarios that point to it. Any question describing "graph-style data" or heavily connected, changing relationships most likely wants Neptune, because AWS expects its own managed service for a matching workload.

## The Graph Data Model

A graph has two building blocks:

- **Nodes** are the "things" in the database: people, places, cards, devices. A node can have a **label** (its type) and **properties** (key-value pairs), such as `name` and `date_of_birth` for a person or an address for a house.
- **Relationships** (edges) connect two nodes. They are **directional** and **named**, for example `:LIVES_AT` or `:BELONGS_TO`. They can be added or removed freely, and a query can follow them in either direction.

![Graph data model example from the lecture: person nodes bob, julie, and bobgoblin, a cat named whiskers, a dog named woofie, two address nodes (742 Evergreen Terrace and Lower Floor, Bridge Street), and a credit card node cc00001, each with key-value properties; named, directional relationships connect them, including LIVES_AT, VISITED, MINION, ENEMY, and BELONGS_TO, with the credit card belonging to both bob and bobgoblin](../assets/neptune-graph-data-model.png)

In the example, whiskers has two `:MINION` relationships (bob and julie) and one `:ENEMY` relationship (woofie). Any node can gain new relationships at any time without a schema change.

### Why not a relational database?

A relational database can store relationships as foreign keys and join tables, but they are second-class citizens:

1. They are usually designed in advance and are awkward to change once the schema is in use.
2. Every hop in a question ("people who share a card with someone who lives at an address that…") becomes another join. Deep or variable-length relationship queries are expensive to run and hard to write, and they get worse at scale.

A graph engine is designed around following relationships from one node to its neighbours, so a multi-hop question becomes a **traversal** instead of a chain of joins.

For example, the credit card `cc00001` in the diagram belongs to both bob and bobgoblin, who live at different addresses. That is a classic fraud signal. In openCypher, one of Neptune's query languages, the question "which cards are shared by people at different addresses?" is a single pattern:

```cypher
MATCH (p1:Person)<-[:BELONGS_TO]-(card:CreditCard)-[:BELONGS_TO]->(p2:Person),
      (p1)-[:LIVES_AT]->(a1:Address),
      (p2)-[:LIVES_AT]->(a2:Address)
WHERE p1.name < p2.name AND a1 <> a2
RETURN card.name, p1.name, a1.address, p2.name, a2.address
```

The `p1.name < p2.name` filter stops each pair from being returned twice in reverse order.

### Graph models and query languages

Neptune Database supports two graph models and [three query languages](https://docs.aws.amazon.com/neptune/latest/userguide/intro.html):

| Graph model                                                                    | Query languages                                                            | Typical use                                                              |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Property graph** (nodes and edges with properties)                           | **Gremlin** (Apache TinkerPop) and **openCypher**, both over the same data | Application graphs: social, fraud, recommendations, network dependencies |
| **RDF** (W3C Resource Description Framework, subject–predicate–object triples) | **SPARQL**                                                                 | Knowledge graphs and linked or standards-based data                      |

The exam does not test the query languages. Recognizing "Gremlin", "openCypher", "SPARQL", or "RDF" in a scenario is enough to identify Neptune.

## Architecture

The lecture describes Neptune as "like RDS or Aurora, but for graph data". Architecturally it is closest to [Aurora](aurora.md#shared-cluster-volume):

- **Cluster:** one primary instance that writes, plus up to **15 read replicas**. A cluster can have up to [15 replicas](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-storage.html), and they are also the **failover targets** if the primary fails.
- **Shared cluster volume:** all instances use one storage volume that keeps [six copies of the data across three AZs](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-storage.html), whether or not you add replicas. Its maximum size is 128 TiB.
- **Multi-AZ** means placing replicas in other AZs so a replica can take over as primary. Storage is already spread across three AZs either way.
- **Read scaling:** add replicas to serve more read queries.

> [!warning] Gotcha: "like RDS" refers to how you use it, not how it stores data
> Neptune is managed like RDS: you pick instance sizes, it runs in your VPC, and it takes automated backups. But its replicas share the cluster volume, as in Aurora. They are not separate copies that receive replicated changes, as in [RDS read replicas](rds.md#read-replicas).

### Networking and access

Neptune runs **inside a VPC** and is **private by default**. Since engine version 1.4.6.x, an instance can have a [public endpoint](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-public-endpoints.html), but this is an **opt-in, per-instance** setting. The cluster must use **IAM database authentication**, so every request is signed with AWS Signature Version 4, and the security group still controls which IP addresses can connect. See [Networking](../concepts/networking.md) and [Identity and Access](../concepts/identity-and-access.md).

### Backups

Like RDS and Aurora, Neptune takes **continuous, incremental backups** to S3 and supports **point-in-time recovery** to any moment in the retention period, which is [1 to 35 days](https://docs.aws.amazon.com/neptune/latest/userguide/backup-restore.html) (default 1 day). Automated backups cannot be turned off, and they are deleted with the cluster unless you take a manual snapshot. Neptune is also one of the databases [AWS Backup](aws-backup.md) can protect centrally.

## Deployment Options and Capabilities

The lecture names three capabilities beyond a provisioned cluster:

| Capability                  | What it is                                  | Key facts                                                                                                                                                                                                                                                                                                            |
| --------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Neptune Serverless**      | Compute that scales automatically with load | Capacity is set in **Neptune Capacity Units (NCUs)**. One NCU is about [2 GiB of memory plus matching CPU and networking](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-serverless-capacity-scaling.html). The range is 1 to 128 NCUs, and it does not scale to zero.                                 |
| **Neptune Global Database** | Cross-Region replication of one cluster     | [One primary Region and up to five read-only secondary Regions](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-global-database.html), each with up to 16 reader instances. Replication happens at the storage level and is typically under one second behind. Only the primary accepts writes.         |
| **Neptune Analytics**       | A separate, memory-optimized graph engine   | Loads a graph from S3 or from a Neptune Database cluster or snapshot and runs [graph algorithms, fast analytic queries, and vector search](https://docs.aws.amazon.com/neptune-analytics/latest/userguide/what-is-neptune-analytics.html) over it in memory, using openCypher. It is not the transactional database. |

Neptune Global Database has two failover paths, as with [Aurora Global Database](aurora.md#aurora-global-database). A [managed planned failover](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-gdb-disaster-recovery.html) moves the primary to another Region with no data loss, for maintenance or relocation. During a Regional outage you instead **detach and promote** a secondary, which can lose writes up to the replication lag at that moment.

Neptune Database and Neptune Analytics serve different jobs:

- **Neptune Database** is the operational store. The application reads and writes it continuously, one relationship at a time.
- **Neptune Analytics** is for heavy analysis of a whole graph, such as PageRank, community detection, shortest paths, or similarity search. It is also the graph and vector store that [Amazon Bedrock Knowledge Bases uses for GraphRAG](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html).

This split mirrors the [OLTP versus OLAP](redshift.md#oltp-vs-olap) split between Aurora and Redshift.

## Use Cases

| Use case                      | Relationships that matter                                                                                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Social media**              | Follow, unfollow, friend, and unfriend links that change constantly; "friends of friends" queries                                                                       |
| **Fraud prevention**          | Who lives at which address and who uses which card, phone, or identity number; shared details across people who should be unrelated, as in the openCypher example above |
| **Recommendation engines**    | What similar or connected people bought, watched, or liked                                                                                                              |
| **Network and IT operations** | Dependencies between systems, so you can trace what an outage affects                                                                                                   |
| **Biology and life sciences** | Drug-to-drug interactions and which treatments worked for which conditions                                                                                              |
| **Knowledge graphs**          | Linking entities and concepts across an organization's data (master data), and giving generative AI applications relationship context through **GraphRAG**              |

GraphRAG (graph-based retrieval-augmented generation) gives a model the relationships around the facts it retrieves, not only the text passages that match a question. This helps it answer questions that need several connected facts. When retrieval only needs to match by meaning, without relationships, a vector store such as [OpenSearch](opensearch.md#use-cases) vector search is the usual choice.

## Choosing Neptune Among AWS Databases

| Data and access pattern                                                                        | Service                                                                                     |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Relational data, fixed schema, SQL joins and transactions                                      | [RDS](rds.md) or [Aurora](aurora.md)                                                        |
| Key-value or document items, single-digit-millisecond reads at any scale, access by known keys | [DynamoDB](dynamodb.md) (see [NoSQL access patterns](../concepts/nosql-access-patterns.md)) |
| In-memory cache or session store in front of another database                                  | [ElastiCache](elasticache.md)                                                               |
| Analytics and reporting over large historical data (OLAP)                                      | [Redshift](redshift.md) or [Athena](athena.md)                                              |
| Highly connected data where queries follow relationships                                       | **Neptune**                                                                                 |

## Exam Quick Reference

| Scenario wording                                                                          | Answer                    |
| ----------------------------------------------------------------------------------------- | ------------------------- |
| "Graph", "graph-style data", "nodes and relationships"                                    | Neptune                   |
| Social network, friends of friends, followers                                             | Neptune                   |
| Fraud detection by linking people, addresses, cards, and devices                          | Neptune                   |
| Recommendation engine based on connections between users and products                     | Neptune                   |
| Gremlin, openCypher, SPARQL, or RDF                                                       | Neptune                   |
| Graph database that scales capacity automatically                                         | Neptune Serverless (NCUs) |
| Graph database with low-latency reads in other Regions or cross-Region DR                 | Neptune Global Database   |
| Run graph algorithms or vector search over a large graph in memory; GraphRAG with Bedrock | Neptune Analytics         |
| Key-value lookups at massive scale                                                        | DynamoDB, not Neptune     |

> [!tip] Exam Tip
> Treat Neptune as "Aurora-style managed database for graph data". It is private in a VPC by default, Multi-AZ through replicas, scales reads with replicas, and supports point-in-time recovery. The deciding factor on the exam is almost always the **data model**: if the scenario is about relationships, choose Neptune.

## References

- Adrian Cantrill, *AWS Certified Solutions Architect – Associate* course: Amazon Neptune lecture
- [AWS: What is Amazon Neptune?](https://docs.aws.amazon.com/neptune/latest/userguide/intro.html)
- [AWS: Neptune storage, reliability, and availability](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-storage.html)
- [AWS: Neptune public endpoints](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-public-endpoints.html)
- [AWS: Neptune backup and restore](https://docs.aws.amazon.com/neptune/latest/userguide/backup-restore.html)
- [AWS: Capacity scaling in Neptune Serverless](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-serverless-capacity-scaling.html)
- [AWS: Neptune global database](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-global-database.html) and [disaster recovery](https://docs.aws.amazon.com/neptune/latest/userguide/neptune-gdb-disaster-recovery.html)
- [AWS: What is Neptune Analytics?](https://docs.aws.amazon.com/neptune-analytics/latest/userguide/what-is-neptune-analytics.html)
- [AWS: Build a knowledge base with graphs (Bedrock GraphRAG)](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html)
- [AWS Database Blog: Build graph applications faster with Neptune public endpoints](https://aws.amazon.com/blogs/database/build-graph-applications-faster-with-amazon-neptune-public-endpoints)
