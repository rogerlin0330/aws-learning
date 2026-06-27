# RDS Data Security

## Overview

RDS data security splits into four distinct concerns. Keep them separate — the exam likes to test whether you can tell authentication apart from authorization, and encryption-in-transit apart from encryption-at-rest.

|Concern|What it covers|
|---|---|
|**Authentication**|How users prove who they are when logging in|
|**Authorization**|What a logged-in user is allowed to do|
|**Encryption in transit**|Protecting data moving between client ↔ RDS|
|**Encryption at rest**|Protecting data written to disk|

---

## Encryption in Transit

- Supported across **all RDS engines**.
- Data between client and RDS instance is encrypted via **SSL/TLS**.
- Can be set to **mandatory on a per-user basis**.

🎯 Trigger keyword: _"force/require SSL"_ or _"encrypt connection"_ → per-user mandatory SSL/TLS.

---

## Encryption at Rest

Two distinct mechanisms. Knowing **who does the encrypting** is the whole game.

### 1. KMS + EBS Encryption (the default/standard method)

This is supported by default across the RDS engines and is handled **below the database engine**.

- The **RDS host** (the server the instance runs on) performs encryption/decryption against the **underlying EBS storage**.
- As far as the database engine is concerned, it's writing **plaintext** to storage — it has **no encryption awareness**. The host encrypts the data before it reaches EBS.
- Uses **KMS**: you select a **CMK** (Customer Master Key) — either _customer-managed_ or _AWS-managed_.
- The CMK generates **DEKs** (Data Encryption Keys), which do the actual bulk encryption work. DEKs are loaded onto the hosts as needed.
- **Coverage:** storage, logs, snapshots, and replicas are all encrypted with the **same CMK**.

> 📚 **Terminology note (AWS docs):** AWS now generally calls these **KMS keys** rather than "CMK." The course term _CMK_ maps to a **customer managed key** or the **AWS managed key (`aws/rds`)**. If you don't specify a customer managed key at creation, RDS defaults to the AWS managed key. ([AWS RDS Encryption docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html))
> 
> 📚 RDS encryption uses the **AES-256** algorithm. _(Added — not stated in the video. Source: [AWS RDS Encryption docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html))_

⚠️ **Encryption cannot be removed once added.** Once an instance is encrypted, it stays encrypted.

⚠️ **You can only enable encryption at instance creation — not afterward.** The video says encryption "cannot be removed once added"; the corollary the exam tests harder is that you **can't add it to an existing unencrypted instance directly** either. The workaround is: snapshot the unencrypted instance → **copy the snapshot with encryption enabled** → restore from the encrypted copy. _(Added for exam completeness — the video does not mention the workaround. Source: [AWS RDS Encryption docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html))_

⚠️ **Read replica encryption is all-or-nothing relative to the source:** you can't have an encrypted replica of an unencrypted instance, or vice versa. _(Added — not in the video. Source: [AWS docs](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/automatically-remediate-unencrypted-amazon-rds-db-instances-and-clusters.html))_

### 2. TDE — Transparent Data Encryption (engine-native)

- Only **Microsoft SQL Server** and **Oracle** support TDE.
- Encryption is **handled inside the database engine itself**, _not_ by the host.
- Because the engine encrypts data **before it leaves the instance**, the trust boundary shrinks — data is secure from the moment the engine writes it.

🎯 **TDE is engine-restricted — MySQL, MariaDB, and PostgreSQL do NOT support TDE.** TDE is native engine encryption, so it only exists where the underlying engine ships it. MySQL/MariaDB/PostgreSQL still get full **at-rest protection via the KMS + EBS (host-side)** path above — they just can't do _engine-native_ TDE. (In the diagram, this is exactly why **Oracle sits on the TDE path** while **MySQL sits on the green host-side KMS arrows**.)

> ⚠️ **Edition nuance — sources disagree, flagging rather than resolving:**
> 
> - **Oracle:** TDE requires the **Oracle Advanced Security** option on **Oracle Enterprise Edition**.
> - **SQL Server:** the main [AWS RDS Encryption docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html) and the [SQL Server TDE setup guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.SQLServer.Options.TDE.html) historically scope TDE to **Enterprise Edition**, but the current [RDS Security features page](https://aws.amazon.com/rds/features/security/) lists **Enterprise _and_ Standard Edition**. ⚠️ Treat "SQL Server Enterprise Edition" as the safe exam answer, but be aware Standard may now qualify. _(Added — not in the video. The course only says "Microsoft SQL and Oracle" without edition detail.)_
> 
> 📚 SQL Server TDE on RDS is enabled via an **option group** (`TRANSPARENT_DATA_ENCRYPTION` / `TDE` option) — a persistent option. _(Added for context; mechanism not covered in the video. Source: [SQL Server TDE docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.SQLServer.Options.TDE.html))_

### 3. RDS Oracle + TDE via CloudHSM (strongest key control)

- **RDS Oracle** can additionally use **CloudHSM** for key services with TDE.
- **CloudHSM is managed entirely by you, with no key exposure to AWS.**
- This enables a trust chain that **does not involve AWS** — valuable for demanding regulatory scenarios.

🎯 **Exam shortcut:** _"no AWS in the chain of trust"_ / _"AWS must have zero key access"_ → **RDS Oracle + TDE + CloudHSM**.

---

## Diagram Walkthrough — KMS Encryption & TDE

![[rds-kms-encryption-vs-tde.png]]

**How to read it** — work _left → middle → right_:

- **Context (top-left text):** With **RDS Oracle**, keys can come from **CloudHSM**, removing AWS from the chain of trust.
- **Left object — CloudHSM (red HSM icon):** connected to the Oracle instance by a **blue arrow**. The blue arrow = the CloudHSM ↔ Oracle key relationship for TDE.
- **Middle — two HOST boxes (orange):** each host runs RDS instances. Left host runs **Oracle** + a **MySQL**; right host runs another **MySQL**. The small key-lock icon in each host's corner = DEKs resident on the host.
- **Green double arrows** (instance ↔ EBS): "Data is **encrypted and decrypted by the host**." This is the KMS/host-side path — the green vertical arrows show the host doing the work between the MySQL engines and EBS volumes.
- **Red double arrow between the two MySQL instances:** replica traffic — **also encrypted**, using the same key.
- **EBS Volumes (green strip, bottom-middle):** the final encrypted-at-rest storage.
- **Right object — KMS (red key icon):** the **red arrows** show **DEKs being loaded onto the hosts as required**. KMS provides AWS- or Customer-managed **CMKs**, which generate the **DEKs**.
- **Footer-right — snapshot/bucket icon:** snapshots of encrypted instances are **also encrypted with the same key**.
- **Bottom-left text:** reiterates TDE is **native DB-engine encryption** — data encrypted _before leaving the instance_ (contrast with the green host-side arrows for KMS).

🎯 The visual contrast to lock in: **TDE path = encryption inside the engine** (Oracle, left); **KMS path = encryption by the host** (green arrows, MySQL).

---

## IAM Authentication for RDS

**Normal RDS logins** use **local database users** (their own username/password). These are **not** IAM users and live **outside AWS's control** — one local user is created when you provision the instance, and that's it.

**IAM authentication** lets you log in using a **short-lived AWS token instead of a password**. Mechanics:

1. On the RDS instance, create a **local DB user account configured to authenticate via an AWS authentication token**.
2. Attach a **policy** to an **IAM user or role** (e.g. an EC2 instance role). The policy **maps that IAM identity to a local RDS database user**.
3. The identity calls **`GenerateDBAuthToken`** (CLI: `generate-db-auth-token`), which works across IAM + RDS.
4. This produces a **token valid for 15 minutes**, usable to log in to the mapped DB user **with no password**.

⚠️ **This is authentication ONLY — not authorization.** 🎯 This is _the_ exam trap for this topic. IAM gets you _in the door_; once inside, **permissions are still controlled by the local DB user** inside the engine. IAM is never used to authorize RDS actions here.

Also note: IAM authentication only applies **if you explicitly enable it** on the instance.

---

## Diagram Walkthrough — IAM Authentication

![[rds-iam-authentication.png]]

**How to read it** — _left → right_, then follow the arrow colors:

- **Top-left text (context):** **Authorization is controlled by the DB Engine** — permissions assigned to the local DB user. _"IAM is NOT used to authorise, only for authentication."_
- **Left object — RDS (blue icon):** has a **local DB account** (badge icon below it) configured to use an **AWS Authentication Token**.
- **Green arrow (RDS ↔ user, middle):** labeled **"DBUser & Token"** — the user presents the DB username + token to RDS to log in. Green = the successful authenticated login path.
- **Right-middle — the human (IAM User) and the chip (EC2 Role):** these are the IAM identities. Each has a **checklist (policy)** above it; the **red double-arrow** at top-right shows the **policy maps the IAM identity onto the local RDS user**.
- **Blue arrows (bottom, flowing right → IAM → back):** the **`generate-db-auth-token`** call. Identities hit **IAM** to generate the **token** (key+clock icon = 15-minute validity). The token then becomes the credential used in the green login path.
- **Right object — IAM (red icon):** the authority that issues the token based on the attached policy.

🎯 Arrow-color summary: **green = authenticated DB login (DBUser + token)**; **blue = token generation via IAM**; **red = policy-to-local-user mapping**.

---

## Implementation Examples

> 📚 **This whole section is supplementary** — none of it is in the course video. It's added for practical reference (your own application code), not exam coverage. The SAA exam tests the _concepts_ above, not SDK syntax. Sources are AWS SDK docs, cited inline. _(Added — flagged per review convention.)_

### Prerequisites (common to all examples)

Three things must be true before any code works:

1. **IAM DB authentication is enabled on the RDS instance** (a toggle on the instance — no downtime to enable). ⚠️ If it's off, every token will be rejected.
2. **A local DB user exists, bound to the AWS auth plugin.** For RDS MySQL:
    
    ```sql
    CREATE USER 'app_iam_user' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app_iam_user';
    ```
    
    🎯 Note the `GRANT` — that's **authorization, handled by the DB engine**, exactly as the note stresses. IAM never grants these.
3. **An IAM policy** allowing `rds-db:connect` on the specific DB user resource is attached to the identity:
    
    ```json
    {  "Version": "2012-10-17",  "Statement": [{    "Effect": "Allow",    "Action": "rds-db:connect",    "Resource": "arn:aws:rds-db:us-east-1:111122223333:dbuser:db-ABCDEFGHIJKL01234/app_iam_user"  }]}
    ```
    

⚠️ **Two non-obvious traps that bite everyone:**

- **SSL/TLS is mandatory** for IAM auth (the token is sent to the DB as a cleartext password; without TLS the server rejects it). Always supply the RDS CA bundle.
- **The token endpoint must be the real DB instance endpoint** — you can't generate a token against a custom Route 53 record. ([AWS docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.Python.html))

---

### How the SDK finds AWS credentials — the Default Credentials Provider Chain

This is the part most people miss: **`GenerateDBAuthToken` itself needs AWS credentials to sign the token.** The SDK doesn't ask you for them — it walks a **default provider chain** and uses the _first_ source that resolves. Understanding the order is what makes "works on my laptop, works on EC2, no code change" possible.

**Order the chain checks (AWS SDK, Java v2 / boto3 — same concept, near-identical order):**

|#|Source|Typical use|
|---|---|---|
|1|**Java system properties** (`aws.accessKeyId` / `aws.secretAccessKey`) _(Java only)_|JVM-flag overrides|
|2|**Environment variables** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)|CI, containers, quick local runs|
|3|**Web Identity Token** (`AWS_WEB_IDENTITY_TOKEN_FILE`)|EKS IRSA / OIDC federation|
|4|**Shared profile file** (`~/.aws/credentials`, `~/.aws/config`)|Local dev (`aws configure`)|
|5|**Container credentials** (ECS/Fargate, via `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`)|ECS task role|
|6|**EC2 Instance Profile via IMDS** (Instance Metadata Service)|**App running on EC2** ← your deployment target|

> 📚 Order confirmed against the [AWS SDK for Java 2.x credentials-chain docs](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html) and the [boto3 credential resolution order](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds/client/generate_db_auth_token.html). boto3's order is the same idea (env → shared creds → assume-role → container → IMDS); the Java property step is JVM-specific.

🎯 **The key insight for "deployed on EC2":** you set **none** of items 1–5. You attach an **IAM role to the EC2 instance** (an _instance profile_), and the chain falls through to step 6, pulling **temporary, auto-rotating credentials from IMDS**. Same binary runs locally (resolves at step 4 via your profile) and in production (resolves at step 6 via the instance role) — **zero credential code, zero secrets on disk.** This is the whole point of `DefaultCredentialsProvider` / `DefaultAWSCredentialsProviderChain`.

⚠️ **IMDSv2 note:** modern SDKs use IMDSv2 (session-token based) by default and fall back to IMDSv1 only if allowed. On a hardened EC2 instance with IMDSv1 disabled, ensure the SDK version is current. ([AWS docs](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/ec2-iam-roles.html))

---

### Python (boto3 + PyMySQL)

```python
import boto3
import pymysql
import ssl

ENDPOINT = "appdb.abcdefghijkl.us-east-1.rds.amazonaws.com"  # real instance endpoint, NOT a CNAME
PORT     = 3306
USER     = "app_iam_user"
REGION   = "us-east-1"
DBNAME   = "appdb"
CA_PATH  = "/opt/certs/global-bundle.pem"  # RDS regional/global CA bundle

# boto3.client('rds') with NO explicit credentials → uses the default provider chain.
# On EC2 this resolves to the instance-profile role via IMDS (step 6 above).
rds = boto3.client("rds", region_name=REGION)

def get_connection():
    # Token is signed locally using whatever creds the chain resolved. 15-min validity.
    token = rds.generate_db_auth_token(
        DBHostname=ENDPOINT, Port=PORT, DBUsername=USER, Region=REGION
    )

    ssl_ctx = ssl.create_default_context(cafile=CA_PATH)  # TLS is mandatory

    return pymysql.connect(
        host=ENDPOINT,
        user=USER,
        password=token,                 # the token IS the password
        port=PORT,
        database=DBNAME,
        ssl=ssl_ctx,
        auth_plugin_map={"mysql_clear_password": None},  # token sent as cleartext over TLS
    )

with get_connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT NOW()")
        print(cur.fetchone())
```

⚠️ **Connection pooling caveat:** the 15-min token only matters at the _connection handshake_ — it does **not** expire active connections. But when a pool opens a _new_ connection after the token has expired, you must generate a fresh token. Generate the token _per new connection_ (as above), not once at startup. ([reference](https://medium.com/@kumar.jayanti/secure-rds-mysql-access-with-iam-authentication-in-python-cd0268ae87d1))

---

### Java + Spring Boot + Alibaba Druid

The clean integration point in Druid is a **custom password callback**: Druid calls it each time it needs to (re)open a physical connection, so you return a freshly generated token. This sidesteps the token-expiry-vs-pool problem entirely.

**`pom.xml` dependencies:**

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>rds</artifactId>
    <!-- pulls in RdsUtilities; use a current 2.x version -->
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

**Custom Druid password callback that mints an IAM token on each connection:**

```java
import com.alibaba.druid.pool.DruidPasswordCallback;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.rds.RdsUtilities;

import java.util.Properties;

public class RdsIamPasswordCallback extends DruidPasswordCallback {

    private final RdsUtilities rdsUtilities = RdsUtilities.builder()
            // No explicit credentials → DefaultCredentialsProvider walks the chain.
            // On EC2 this lands on the instance-profile role via IMDS (step 6).
            .credentialsProvider(DefaultCredentialsProvider.create())
            .region(Region.US_EAST_1)
            .build();

    private static final String HOST = "appdb.abcdefghijkl.us-east-1.rds.amazonaws.com";
    private static final int    PORT = 3306;
    private static final String USER = "app_iam_user";

    @Override
    public void setProperties(Properties properties) {
        super.setProperties(properties);
        // Generate a fresh 15-minute token. Druid invokes this when opening a
        // physical connection, so pooled connections always get a valid token.
        String token = rdsUtilities.generateAuthenticationToken(builder -> builder
                .hostname(HOST)
                .port(PORT)
                .username(USER));
        // Druid uses the value set here as the connection password.
        setPassword(token.toCharArray());
    }
}
```

**`application.yml`:**

```yaml
spring:
  datasource:
    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://appdb.abcdefghijkl.us-east-1.rds.amazonaws.com:3306/appdb?useSSL=true&requireSSL=true&verifyServerCertificate=true&trustCertificateKeyStoreUrl=file:/opt/certs/rds-truststore.jks
      username: app_iam_user
      # NO password here — supplied dynamically by the callback below
      password-callback-class-name: com.example.config.RdsIamPasswordCallback
      # TLS is mandatory for IAM auth; point the JDBC URL at the RDS CA truststore
```

> 📚 `RdsUtilities.generateAuthenticationToken(...)` is the **Java SDK v2** API (the v1 equivalent was `RdsIamAuthTokenGenerator`). Source: [RdsUtilities Javadoc](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/services/rds/RdsUtilities.html). The Druid `DruidPasswordCallback` pattern is the standard hook for dynamic passwords.

⚠️ Import the RDS CA bundle into a JKS truststore for the JDBC `useSSL`/`trustCertificateKeyStoreUrl` settings. Without TLS the IAM token is rejected.

---

### Deploying on EC2 — what actually has to be in place

Putting the chain knowledge into a concrete checklist for the "app on EC2" case:

1. **Create an IAM role** with the `rds-db:connect` policy (the JSON in Prerequisites). This is the **EC2 instance role** shown in the IAM Authentication diagram (the chip icon).
2. **Attach it to the EC2 instance** as an instance profile (Console: instance → _Actions → Security → Modify IAM role_).
3. **Deploy the app with no AWS credentials configured** — no env vars, no `~/.aws/credentials`. The SDK's default chain falls through to **IMDS (step 6)** and transparently uses the role's temporary credentials.
4. **Ship the RDS CA bundle** with the app for the TLS connection.

🎯 Result: the EC2 role authenticates to _AWS_ (to sign the token) → the token authenticates to _RDS_ → the **local DB user** authorizes the actual SQL. Three distinct steps, mapping exactly onto the diagram's blue (token gen) and green (DB login) arrows. No long-lived database password exists anywhere.

---

## Exam Quick Reference

|Trigger / Scenario|Answer|
|---|---|
|Encrypt client ↔ RDS connection, per user|**SSL/TLS in transit** (all engines, can be mandatory)|
|Default at-rest encryption, host does the work|**KMS + EBS** (CMK → DEKs; engine unaware)|
|Engine-native at-rest encryption|**TDE** — **SQL Server & Oracle only** (not MySQL/MariaDB/PostgreSQL)|
|"AWS must have no access to keys" / no AWS in chain of trust|**RDS Oracle + TDE + CloudHSM**|
|Log in to RDS without a DB password, using AWS identity|**IAM authentication** (`GenerateDBAuthToken`, 15-min token)|
|"Does IAM control what the user can do in the DB?"|**No** — authorization stays with the local DB user|
|Storage, logs, snapshots, replicas all encrypted|KMS at-rest, **same CMK**|
|Add encryption to an existing unencrypted instance|Snapshot → **copy snapshot with encryption** → restore|

---

## Traps & Gotchas

- ⚠️ **IAM auth ≠ authorization.** It only handles _authentication_; the local DB user still governs permissions. (The single most-tested point here.)
- ⚠️ **Encryption can't be removed**, and **can't be added in-place** after creation — only via the snapshot-copy workaround.
- ⚠️ **TDE is engine-restricted to SQL Server and Oracle only** — **MySQL, MariaDB, and PostgreSQL cannot use TDE** (they still get at-rest protection via KMS/EBS, the broadly-supported default). Don't let "MySQL is encrypted" trick you into "MySQL supports TDE."
- ⚠️ **CloudHSM key option is RDS Oracle-specific** for the "no AWS trust" scenario — don't generalize it to other engines.
- ⚠️ **Token validity is 15 minutes** — short-lived by design.
- ⚠️ Don't confuse **CMK** (the master key) with **DEK** (the key that actually encrypts data). CMK _generates_ DEKs.

---

> 📚 **Sources beyond the course transcript** (flagged per review convention): AWS-256 algorithm, the create-time-only restriction, the snapshot-copy workaround, and the read-replica encryption rule are not stated in the video and were added from official AWS documentation:
> 
> - [Encrypting Amazon RDS resources](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
> - [Automatically remediate unencrypted RDS instances (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/automatically-remediate-unencrypted-amazon-rds-db-instances-and-clusters.html)