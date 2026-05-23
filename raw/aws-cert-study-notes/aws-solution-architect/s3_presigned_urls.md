# S3 Pre-Signed URLs

## Foundational Concept

A **pre-signed URL** is a mechanism that lets you grant another person or application **time-limited access to a specific S3 object** using _your_ credentials — without giving them an AWS identity, sharing credentials, or making the bucket/object public.

### The problem it solves

By default, an S3 bucket is private. To access an object, a requester must:

1. Have an AWS identity (IAM user/role)
2. Authenticate to AWS
3. Be authorized via a permissions policy

Anyone without AWS credentials cannot access the object. If you need to give an external user (or unauthenticated client) access, the three "naive" alternatives are all suboptimal:

|Option|Problem|
|---|---|
|Give the external user an IAM identity|Overkill for short-term/one-off access|
|Share AWS credentials|Bad security practice|
|Make the bucket/object public|Exposes sensitive data|

Pre-signed URLs solve this cleanly: **the URL itself carries pre-authorized, time-limited, object-scoped access.**

---

## Mechanics: How a Pre-Signed URL Is Generated and Used

### Inputs required to generate a pre-signed URL

When an identity requests a pre-signed URL from S3, it must provide:

- **Security credentials** (the generator's credentials)
- **Bucket name**
- **Object key**
- **Expiry date and time**
- **HTTP method** — GET (download) or PUT (upload)

S3 returns a URL with all of this encoded inside it. The URL is bound to:

- The specific bucket and object
- The identity that generated it
- An expiration timestamp

### What happens when the URL is used

When the holder of the URL accesses the object, **S3 treats the request as if it came from the identity that generated the URL** — for that one object, until expiry.

> The masked man (anonymous user) effectively becomes IAM admin for that single object until the URL expires.

Pre-signed URLs support both:

- **GET** → download an object from S3
- **PUT** → upload an object to S3

---

## Architecture Pattern: Application-Mediated Access to Private S3

This is the most common production use case for pre-signed URLs and the one most likely to appear in exam scenarios.

![[s3-presigned-url-flow.png]]

### Scenario

A web application (e.g., a wildlife video platform for _Animals for Life_) hosts large media files. The videos are migrated from the application server into an S3 bucket to leverage S3's strengths for large media storage. The bucket remains **private**.

### The flow (matching the diagram)

| Step                | Action                                                                                                                                       |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Request**      | End user's browser requests a page from the application server (e.g., a wildlife article that includes an embedded video).                   |
| **2. presignedURL** | Application server, authenticated as IAM user `iamapp1`, calls S3 to generate a pre-signed URL for the requested video object.               |
| **3. CreateURL**    | S3 creates the URL — encoded with `iamapp1`'s permissions, scoped to the specific object, with a short expiry (e.g., 2–3 hours).             |
| **4. URL to User**  | Application server returns the pre-signed URL to the user's browser along with the rest of the page content.                                 |
| **5. Download**     | The user's browser uses the pre-signed URL to download the video directly from the media bucket — bypassing the application server entirely. |

### Why this pattern is powerful

- **Bucket stays private** — no public access required
- **No IAM identity per user** — the application owns one IAM user (`iamapp1`); end users never touch AWS
- **App server is offloaded** — large media downloads go directly browser ↔ S3, not through the application server
- **Common in serverless architectures** — where there is no thick app server to broker access

> The video transcript references this exact pattern: "Pre-signed URLs are often used when you offload media into S3 or as part of serverless architectures."

---

## Decision Guidance

### IAM user vs. IAM role for the generator

**Always use a long-term identity (IAM user) to generate pre-signed URLs in production application patterns.**

The reasoning ties to a core principle from earlier IAM lessons:

- An **IAM user** is appropriate when you can visualize a fixed, small number of the thing using the identity (e.g., one application = one service account user).
- An **IAM role** provides _temporary_ credentials via STS.

#### Why roles are problematic for pre-signed URLs

```
Role temporary credentials lifetime:  ──────[expires]──────────────
Pre-signed URL validity:              ──────────────────────[expires]
                                              ^
                                              URL stops working here
                                              even though it "looks" valid
```

If you assume a role and use its temporary credentials to generate a pre-signed URL, the URL will likely outlive the credentials. Once the role credentials expire, **the pre-signed URL stops working** — even if its own expiry timestamp hasn't been reached.

**Rule:** Generate pre-signed URLs from IAM users with stable, long-lived credentials. The application's IAM user permissions should change rarely, if ever.

---

## Subtle Behaviors (Exam Power-Ups)

These are the gotchas the transcript explicitly flags as exam-relevant.

### 1. You can generate a pre-signed URL for an object you have no access to

The only requirements to _generate_ a URL are:

- Specify a bucket + object key
- Specify an expiry date/time

S3 will produce the URL regardless of whether the generating identity has permission to the object. **The URL just won't work** when used, because it inherits the generator's (lack of) permissions.

> There are very few legitimate use cases for this, but it's a behavior the exam may test.

### 2. Permissions are evaluated _at the time the URL is used_, not when it was generated

This is the most important nuance:

> When the URL is used, the permissions match the **current** permissions of the identity that generated it — not the permissions at the time of generation.

**Implications when you see `AccessDenied` on a pre-signed URL:**

|Possible cause|Explanation|
|---|---|
|Generator never had access|URL was created but was always non-functional|
|Generator's permissions were revoked|URL worked initially, then broke when policy changed|
|Generator was an IAM role and credentials expired|The temporary credentials backing the URL are no longer valid|
|URL has expired|Past the expiry timestamp encoded in the URL|

For an IAM user with static permissions, this is rarely a problem in practice — the permissions don't change between URL generation and use. **This is exactly why IAM users (not roles) are preferred for this pattern.**

### 3. Pre-signed URLs are self-contained

All authentication information is **encoded inside the URL**. No separate credentials need to be transmitted when using the URL — the URL _is_ the credential.

---

## Exam Cheat Sheet

|Keyword / Scenario|Answer|
|---|---|
|Give external user time-limited access to a single S3 object|**Pre-signed URL**|
|Private S3 bucket + need to share one object without making bucket public|**Pre-signed URL**|
|Direct browser-to-S3 upload/download from a web application|**Pre-signed URL** (PUT for upload, GET for download)|
|Offload large media from app server to S3 while keeping access controlled|**Pre-signed URL** pattern|
|Serverless app needs to broker S3 access without a thick app server|**Pre-signed URL**|
|URL works briefly then returns AccessDenied|Generator's permissions changed, **or** generator was a role and STS credentials expired|
|URL returns AccessDenied immediately|Generator never had permission to the object|
|"Should I use an IAM user or role to generate pre-signed URLs?"|**IAM user** (long-term credentials)|
|Can I generate a pre-signed URL for an object I don't have access to?|**Yes** — generation succeeds, but the URL won't work when used|
|When are pre-signed URL permissions evaluated?|At the time of **use**, against the **current** permissions of the generator|
|Supported operations|**GET (download)** and **PUT (upload)**|
|What's encoded in the URL?|Bucket, object key, generator identity, expiry, signature|

---

## Key Takeaways

1. **Pre-signed URLs grant time-limited, object-scoped access** to a private S3 object using the generator's credentials.
2. **The URL holder effectively _is_ the generator** for that one object until expiry — same permissions, same identity context.
3. **Always generate from IAM users, not roles** — roles' temporary credentials can expire before the URL does, breaking the URL.
4. **Permissions are evaluated at use time, not generation time** — revoking the generator's access invalidates outstanding URLs.
5. **Supports both downloads (GET) and uploads (PUT)** — same mechanism, different HTTP methods.
6. **Common architecture pattern**: app server (as IAM user) generates URL → returns to browser → browser interacts directly with S3, bypassing the app server for large media transfer.