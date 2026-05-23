# S3 Access Logs

## Foundational Concept

**S3 Access Logging** is a feature that provides visibility into requests made against an S3 bucket. It captures detailed records of bucket and object access — useful for security auditing, compliance, billing investigation, and understanding customer access patterns.

The model involves two buckets:

- **Source Bucket** — the bucket you want visibility into. Logging is enabled _on this bucket_.
- **Target Bucket** — the destination bucket where log files are delivered.

![[s3-access-logs.png]]

---

## Mechanics

### Enabling Access Logging

Logging is configured **on the source bucket** via one of two methods:

1. **Console UI** — toggle access logging in bucket properties
2. **API/CLI** — `PUT Bucket logging` operation

The configuration is an XML document that specifies the target bucket and an optional prefix:

```xml
<BucketLoggingStatus xmlns="http://doc.s3.amazonaws.com/2006-03-01">
  <LoggingEnabled>
    <TargetBucket>targetcats</TargetBucket>
    <TargetPrefix>catlogslogs/</TargetPrefix>
  </LoggingEnabled>
</BucketLoggingStatus>
```

### The S3 Log Delivery Group

Log delivery is handled by an AWS-managed system called the **S3 Log Delivery Group**, which:

- Reads the logging configuration set on the source bucket
- Captures access events against the source bucket
- Writes log files into the target bucket

**Best-efforts delivery** — this is not real-time. Enabling logging or making configuration changes can take **a few hours** to take effect, and log delivery itself is not guaranteed to be immediate or complete.

### Granting Permission to the Target Bucket

The Log Delivery Group needs **write access to the target bucket**. This is granted via an **ACL** (not a bucket policy) on the target bucket — specifically, by adding the **S3 Log Delivery Group** principal with write permission.

> ⚠️ This ACL-based mechanism is one of the few remaining legitimate uses of S3 ACLs in modern accounts (where bucket policies are otherwise preferred).

---

## Log File Format

Log files in the target bucket consist of multiple **log records**:

|Level|Delimiter|
|---|---|
|Records within a file|**newline-delimited**|
|Attributes within a record|**space-delimited**|

The format is conceptually similar to an **Apache access log**. Each record contains attributes such as:

- Date and time of the request
- Requester identity
- Operation performed (GET, PUT, DELETE, etc.)
- HTTP status code
- Error code (if any)
- Bytes transferred, request URI, referrer, user agent, and more

### Multiple Source Buckets → One Target Bucket

A single target bucket can receive logs from **many source buckets**. They are separated using the **TargetPrefix** value in each source bucket's logging configuration (e.g., `catlogslogs/`, `dogbucketlogs/`).

---

## Use Cases

- **Security auditing** — who accessed what, when, and from where
- **Access audits** — compliance evidence
- **Access pattern analysis** — understand customer behavior
- **Billing investigation** — correlate request volume with S3 charges

---

## Lifecycle Management

S3 Access Logs come with **no built-in lifecycle management**. You are personally responsible for:

- Transitioning old log files to cheaper storage classes (e.g., Glacier)
- Deleting log files after a retention period

This is typically handled by attaching an **S3 Lifecycle Configuration** to the target bucket.

---

## Decision Guidance

|Question|Answer|
|---|---|
|Where do I enable logging?|On the **source bucket**|
|Where do logs land?|In the **target bucket** under the configured prefix|
|How do I grant log delivery permission?|**ACL** on the target bucket allowing the **S3 Log Delivery Group** write access|
|Is delivery guaranteed and real-time?|**No** — best-efforts, with delays up to a few hours|
|Can one target bucket serve many source buckets?|**Yes** — separate them with prefixes|
|Who manages log retention?|**You do** — via lifecycle rules|

### S3 Access Logs vs. CloudTrail Data Events

|Capability|S3 Access Logs|CloudTrail Data Events|
|---|---|---|
|Delivery model|Best-efforts (hours)|Near real-time|
|Cost|Storage cost of logs only|Per-event charge|
|Detail level|Apache-style records|Structured JSON, full IAM context|
|Integration|Manual (Athena/parse)|Native CloudWatch, EventBridge, Athena|
|Permission model|ACL on target bucket|CloudTrail-managed|

In modern setups, CloudTrail data events are typically preferred for security workflows; S3 Access Logs remain relevant for cost analysis and legacy/simple visibility needs.

---

## Exam Cheat Sheet

|Keyword / Scenario|Answer|
|---|---|
|Visibility into requests to a bucket|**S3 Access Logging**|
|Where logging is enabled|**Source bucket**|
|Where logs are written|**Target bucket**|
|How to grant log delivery permission|**ACL** granting **S3 Log Delivery Group** write access|
|API operation to enable|**PUT Bucket logging**|
|Delivery guarantee|**Best efforts**, may take **a few hours**|
|Record delimiter within a file|**Newline**|
|Attribute delimiter within a record|**Space**|
|Multiple sources, one target|Use **TargetPrefix** to separate|
|Log retention/cleanup|**Your responsibility** — use S3 Lifecycle rules|
|Comparable log format|**Apache access log** style|