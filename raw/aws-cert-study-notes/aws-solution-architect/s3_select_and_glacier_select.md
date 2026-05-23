# S3 Select & Glacier Select

> Server-side filtering features that let applications retrieve **partial objects** from S3 or Glacier using a SQL-like expression, rather than downloading the entire object and filtering client-side.

---

## Foundational Concept

Both S3 and Glacier are **massively scalable** object stores. S3 supports objects up to **5 TB** and an effectively unlimited number of them. By default, when an application reads an object, it receives the **entire object** — there is no native partial-read filter on the storage side.

This creates two problems when you only need a slice of the data:

1. **Cost** — you pay for the full object's data transfer, even if you discard 99% of it on the client.
2. **Time** — the full object must be loaded and streamed before the client can begin filtering.

Client-side filtering "fixes" nothing economically: the bytes have already been transferred and billed by the time your application sees them. The filter just throws bytes away after the fact.

**S3 Select** and **Glacier Select** push the filter **into the storage service itself**, so only the matching subset is ever transferred to the application.

---

## Mechanics

The application supplies a **cut-down, SQL-like expression** to the S3 Select or Glacier Select API. The service applies that expression to the raw object data inside S3/Glacier and returns only the matching rows/fields to the client.

### Supported formats

- CSV
- JSON
- Parquet (S3 Select)
- BZIP2 compression is supported for **CSV** and **JSON**

### What happens architecturally

|Aspect|Without S3 Select|With S3 Select|
|---|---|---|
|Where filtering happens|In the application (client-side)|Inside the S3 service (server-side)|
|Data transferred|Entire object|Pre-filtered subset only|
|Data billed|Entire object|Pre-filtered subset only|
|Time to usable result|Full object download + filter|Filtered stream only|
|Performance ceiling|Limited by full-object transfer|Up to **400% faster**|
|Cost ceiling|Full-object retrieval cost|Up to **80% cheaper**|

### Visual comparison

![[s3-w-wo-glacier-select-comparison.png]]

The diagram shows the same application reading the same bucket two ways:

- **Top (without Select)**: S3 streams the full object out to the application. The funnel (filter) sits inside the app — data is already billed and transferred before anything is discarded.
- **Bottom (with Select)**: The SQL-like expression is provided to S3 Select. The funnel sits **inside the S3 service**, next to the raw data. Only the pre-filtered result crosses the wire to the app.

---

## Decision Guidance

**Reach for S3 Select / Glacier Select when:**

- The application only needs a subset of rows or columns from large structured objects (CSV, JSON, Parquet).
- Data transfer cost or latency is a meaningful constraint.
- The objects are large enough that filtering client-side is wasteful (e.g. multi-GB CSV/JSON exports where the app uses one column).
- Storage format is already one of the supported types — no schema-on-write or query-engine setup required.

**Don't reach for S3 Select / Glacier Select when:**

- The application genuinely needs the full object (image, video, archive, full backup restore).
- The objects are small enough that the per-request overhead dominates.
- You need **joins, aggregations across many objects, or a real query layer** — that's **Athena's** job, not S3 Select's. S3 Select operates on **one object at a time**.
- The object format isn't supported (e.g. arbitrary binary, proprietary formats).

**Note**: Glacier Select operates against objects in the Glacier storage class. Like other Glacier retrievals, it has retrieval-tier semantics — the "select" is a retrieval job, not an instant query.

---

## Exam Cheat Sheet

|If the question mentions…|Think…|
|---|---|
|"Retrieve **part of** an object" / "filter at the source" / "reduce data transferred from S3"|**S3 Select**|
|Same as above but the data lives in **Glacier**|**Glacier Select**|
|**SQL-like expression** applied to a single object|**S3 Select / Glacier Select**|
|Querying **across many objects** in S3, joins, aggregations, schema catalog|**Athena**, not S3 Select|
|Filtering happens **client-side / in-app** after full download|This is the **anti-pattern** S3 Select fixes|
|Supported formats: **CSV, JSON, Parquet**, with **BZIP2** for CSV/JSON|S3 Select compatibility|
|"Up to **400% faster, 80% cheaper**"|S3 Select / Glacier Select marketing numbers|
|Application needs the **entire** object|Plain S3 GET — Select adds no value|

---

## Key Takeaway

S3 Select and Glacier Select move the filter **from the application to the service**. The architectural win is that the network only carries the bytes the application actually needs, which makes retrieval both faster and cheaper — but the feature is **per-object** and is **not** a substitute for a query engine like Athena when the workload spans many objects.