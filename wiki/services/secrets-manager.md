---
title: AWS Secrets Manager
type: service
tags: [secrets-manager, credentials, rotation, iam, kms, lambda, rds, postgresql, spring-boot, druid, jdbc, connection-pool]
certs: [aws-saa]
updated: 2026-11-14
---

# AWS Secrets Manager

Secrets Manager stores credentials and supports automatic rotation. For a database secret, rotation must update both the secret stored in Secrets Manager and the password in the database. Applications still need a way to obtain the updated credentials when connecting.

The [identity and access](../concepts/identity-and-access.md) distinction matters here: an application's [IAM role](iam.md) authorizes retrieval from Secrets Manager; the retrieved database username and password authenticate a separate database connection.

## What Secrets Manager Does

Secrets Manager manages the lifecycle of **database passwords, API keys, application credentials, and tokens**. Applications retrieve a secret at runtime through an AWS SDK instead of embedding it in source code. Operators can also use the console, CLI, or API. It overlaps with [Parameter Store](ssm.md#parameter-store), but its distinguishing features are **automatic rotation and integration with the system that uses the credential**.

A secret can contain text or binary data up to **65,536 bytes (64 KiB)**. Database secrets commonly hold a JSON object containing a username, password, and connection information; rotation templates require particular fields. See the [supported secret structures](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_secret_json_structure.html).

The service is [regional](../concepts/resource-scope.md). You can explicitly replicate secrets into other Regions; rotation occurs on the primary and propagates the new value to its replicas. See [cross-Region replication](https://docs.aws.amazon.com/secretsmanager/latest/userguide/replicate-secrets.html).

### Secret versions and the current-version label

A **secret** has a stable name or ARN that the application uses to find it. Its value can change over time: each update creates a **version** containing a particular value, such as a database username and password. The application can keep using the same secret name while the stored credentials change.

Secrets Manager attaches movable **staging labels** to these versions. **`AWSCURRENT`** is the label identifying the current version of the secret, which is the version that applications should normally use. When the `GetSecretValue` API is called without a version ID or label, it returns the value from the version labeled `AWSCURRENT`. See [secret versions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/whats-in-a-secret.html) and [retrieval defaults](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html).

During rotation, a label called **`AWSPENDING`** points to the candidate version being prepared and tested. When rotation succeeds, the `AWSCURRENT` label is moved to point to the new version, and a label called **`AWSPREVIOUS`** is attached to the formerly current version.

## Architecture: Retrieve, Connect, Rotate

The lecture's **Bin Chicken Bounty** application uses an SDK to retrieve credentials, then connects directly to [RDS](rds.md). The diagram shows the **Lambda rotation** path; managed rotation is covered separately below.

![Secrets Manager architecture: the application uses an SDK and IAM authorization to retrieve a KMS-encrypted secret, then connects to RDS with the retrieved credentials; a separately authorized Lambda function rotates the secret and the database password together.](../assets/secrets-manager-architecture.png)

1. **Retrieve:** application code calls the Secrets Manager SDK for the database secret.
2. **Authorize:** the SDK signs the AWS request using credentials, preferably temporary credentials from the application's [IAM role](iam.md#iam-roles). IAM determines whether it can retrieve that secret. The IAM arrow represents authorization, not a separate IAM network call for every read.
3. **Connect:** the application receives the secret value and uses the database username/password to establish its RDS connection. Secrets Manager is not a database proxy and does not carry SQL traffic.
4. **Rotate:** Secrets Manager invokes the configured [Lambda](lambda.md) rotation function to coordinate a new secret value with the target database credential.
5. **Give the rotator its own permissions:** the Lambda execution role authorizes AWS API operations. The function also needs network connectivity and database credentials with permission to change the relevant database user's password.

These are two distinct authentication steps: **AWS credentials authorize secret retrieval; database credentials authenticate the database session**. Granting `secretsmanager:GetSecretValue` does not itself grant database privileges. The rotation function's execution role is separate from the application's role; an application normally needs to read its secret, not rotate it.

## Encryption and Access Control

Secrets are encrypted at rest through [KMS envelope encryption](kms.md): a data key encrypts the secret value, and a symmetric KMS key protects that data key. Use the AWS managed key `aws/secretsmanager` or a customer-managed symmetric encryption key. [AWS documents the encryption flow and key permissions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/security-encryption.html).

With a **customer-managed KMS key**, reading a secret requires authorization for both Secrets Manager retrieval and `kms:Decrypt` on the key. This supports **separation of duties**: permission to manage or inspect secret metadata need not imply permission to decrypt its value. IAM policies and the KMS key policy must allow the intended access.

> [!warning] Gotcha
> Encryption at rest protects stored data; it does not prevent an authorized application from exposing the plaintext after retrieval. Keep secret values out of application logs. Rotating a KMS key also does **not** rotate a database password: encryption-key rotation and credential rotation solve different problems.

## Automatic Rotation

**Rotation must update both the secret and the target database/service.** Merely editing the password field in Secrets Manager does not change a database user's password. There are two automatic rotation mechanisms, as shown in the table below (also described in the [AWS rotation guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)):

| Mechanism            | Who performs the update?                                                                                     | Typical use                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| **Managed rotation** | The integrating AWS service manages the secret and credential together; no customer Lambda rotation function | [RDS](rds.md) and [Aurora](aurora.md) master-user credentials managed through those services |
| **Lambda rotation**  | A configured Lambda function updates the secret and the target                                               | Application database users and other secrets supported by templates or custom rotation code  |

> [!warning] Gotcha
> Automatic rotation does not automatically refresh a password already loaded into an application's configuration or connection pool. Clients need a retrieval/cache-refresh strategy. The [Spring Boot + Druid example](#example-spring-boot--alibaba-druid--rds-postgresql-auto-rotation) below explains new connections, rotation timing, and existing PostgreSQL sessions.

### Managed rotation

Managed rotation applies only to supported service-managed credentials (i.e., credentials created and owned by the service, e.g. RDS). For RDS/Aurora, that means only the master user: the "Manage master credentials in AWS Secrets Manager" setting in the instance configuration applies to the master user, and RDS does not manage users you create inside the database with SQL. The table below summarizes which mechanism applies:

| Database user                                                                                              | Rotation mechanism                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Master user, with "Manage master credentials in AWS Secrets Manager" turned on in RDS/Aurora configuration | Managed rotation: **RDS creates and owns** the secret (named `rds!db-…`, or `rds!cluster-…` for Aurora). Rotating the password in Secrets Manager triggers RDS to update the master user's password, without the need to use a Lambda function |
| Master user, with a password you set and stored in a secret yourself                                       | Lambda rotation, usually the single-user template                                                                                                                                                                                              |
| Application or other non-master users                                                                      | Lambda rotation, single-user or alternating users                                                                                                                                                                                              |

### Lambda rotation

**For Lambda rotation, AWS supplies the code for supported databases, but the function runs in your account.**

| Secret type                                                        | Where the rotation code comes from                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Supported databases (RDS and Aurora engines, Redshift, DocumentDB) | AWS publishes [rotation function templates](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html); Secrets Manager can create the function from a template when you turn on rotation in the console                                           |
| Other secrets, such as third-party API keys                        | You write the function; AWS's [generic template](https://github.com/aws-samples/aws-secrets-manager-rotation-lambdas/tree/master/SecretsManagerRotationTemplate) provides the [four-step structure](#how-the-single-user-strategy-works), but you implement how the target credential is changed |

Either way, the function is an ordinary [Lambda](lambda.md) function that you pay for and maintain. It needs [network access](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-network-rotation.html) to both the database and the Secrets Manager API; from private subnets, that means a VPC endpoint or a NAT gateway.

#### Deep dive into Lambda rotation for database secrets: single-user vs alternating users

For Lambda rotation, for database secrets, Secrets Manager offers two rotation strategies.

AWS describes single-user rotation as the simplest strategy and suitable for most use cases, while explicitly identifying alternating users as suitable for applications requiring high availability. This is a choice about availability and operational complexity, not a rule that single-user rotation is unsuitable for production. See [AWS's strategy guidance](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html).

| Consideration                      | Single user                                                 | Alternating users                                                                                        |
| ---------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Credential being changed           | The currently published user's password                     | The other user's password, while the current pair remains valid                                          |
| New connections during publication | May need retries across the database/secret mismatch window | Can keep using the currently published pair while the other is prepared                                  |
| Operational responsibility         | Maintain one application user and suitable retries          | Maintain equivalent access for two application users, plus a separate administrative secret for rotation |

For the direct RDS application connection in the example below, prioritizing alternating users is a reasonable engineering choice when new-connection availability during rotation matters. It still requires credential refresh and sensible failure handling; it is not a guarantee against all connection failures. Verify both users' effective permissions, including after permission changes.

##### How the single-user strategy works

Suppose the current version contains password P1 and rotation will replace it with P2. The Lambda function then follows [AWS's four Lambda rotation steps](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_lambda-functions.html):

- **`createSecret`:** generate P2 and store it in a new version labeled `AWSPENDING`. At this point, the database and the version labeled `AWSCURRENT` still use P1.
- **`setSecret`:** change the database password to P2.
- **`testSecret`:** check that P2 can authenticate successfully.
- **`finishSecret`:** only after that check, move `AWSCURRENT` to the version containing P2; the previous current version receives `AWSPREVIOUS`.

Applications retrieving the current version now obtain P2 without changing the secret name in their configuration.

> [!warning] Gotcha
> Because changing the database and moving the label are separate operations, there can be an interval when the database requires P2 but a read of the current secret still returns P1. This is what it means for the updates to be **non-atomic**: they do not take effect together as one indivisible operation.

##### How the alternating-users strategy works

The previous section describes how **single-user rotation** works; it changes the password of the same database user. Another rotation strategy supported by Secrets Manager, called **alternating-user rotation**, maintains two database users and switches the application's current secret between their credential pairs instead. Updating the other user while the current pair remains usable reduces the connection-creation gap. See [AWS rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html).

With the official PostgreSQL template, prepare **one application database user**, for example `app_user`, grant its required access, and store its credentials in the application secret. Also provide a separate secret containing administrative database credentials that the rotation Lambda can use to create users, grant access, and change passwords. The application secret's `masterarn` field identifies that administrative secret; the application itself only needs access to its application secret.

The Lambda then follows this sequence:

- **First rotation:** create `app_user_clone`, set its password, and grant it membership in the original PostgreSQL role. Test the credentials, then move `AWSCURRENT` to the clone's credential version.
- **Second rotation:** update `app_user`'s password, test it, and move `AWSCURRENT` back to that user's new credential version.
- **Third rotation:** update `app_user_clone`'s password, test it, and move `AWSCURRENT` back to the clone's new credential version.
- **Subsequent rotations:** repeat the alternation; no new user is created each time.

**The user being updated is the one that `AWSCURRENT` does not currently select.** Both users' credential versions belong to the same application secret. You only prepare the original application user; the `_clone` suffix and automatic creation of the second user come from the [official PostgreSQL template](https://github.com/aws-samples/aws-secrets-manager-rotation-lambdas/blob/master/SecretsManagerRDSPostgreSQLRotationMultiUser/lambda_function.py), particularly `set_secret()` and `get_alt_username()`.

**The non-atomic gap does not cause login failures with alternating users.** The steps are still separate operations, but the user whose password changes is the one that `AWSCURRENT` does not point to, so the current secret never contains a password the database rejects:

| Step                                          | `AWSCURRENT` points to                    | Can clients using the current secret log in? |
| --------------------------------------------- | ----------------------------------------- | -------------------------------------------- |
| Before rotation                               | User A, password unchanged                | Yes                                          |
| `setSecret` changes user B's password         | Still user A                              | Yes; user A is untouched                     |
| `finishSecret` moves `AWSCURRENT` to user B   | User B, whose new password is already set | Yes                                          |
| A client still uses cached user A credentials | User B                                    | Yes; user A's password is still valid        |

**The previous pair stays valid until the next rotation, not until its sessions close.** Secrets Manager does not track database sessions and does not invalidate user A's password when `AWSCURRENT` moves to user B. At the next rotation, user A becomes the user that `AWSCURRENT` does not select, so the Lambda function changes user A's password regardless of how many user A sessions remain open; those already-authenticated sessions continue, as described in [already-established connections](#2-what-happens-to-already-established-connections-during-rotation). A client fails only if it still uses user A's credentials after that change, that is, if its cached pair is older than one rotation interval. Keeping the client cache TTL much shorter than the rotation interval avoids this, and the JDBC wrapper in the example below refreshes and retries if it happens.

**The configured Secrets Manager schedule drives rotation, and Lambda performs the updates.** Operators can also trigger rotation manually. The administrative account is separate from the two alternating application users. See [rotation scheduling](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_turn-on-for-db.html).

###### Responding to a leaked credential

Because the previous pair stays valid for a full rotation interval, **a single manual rotation does not lock out someone who stole the current credentials.** Suppose user A's password leaks while `AWSCURRENT` points to user A. One rotation updates user B and moves `AWSCURRENT` to user B; user A's leaked password is untouched and remains valid until the following rotation. With the single-user strategy, by contrast, one rotation changes the leaked password directly.

To invalidate the leaked password promptly:

1. **Rotate twice.** The first rotation moves applications to user B; the second changes user A's password. Let the first rotation finish, and give clients time to pick up user B's credentials, before starting the second. Otherwise, clients still using user A hit authentication failures and depend on refresh-and-retry.
2. **End existing sessions if needed.** A password change only blocks new logins; sessions that already authenticated with the leaked password stay open, as described in [already-established connections](#2-what-happens-to-already-established-connections-during-rotation). In PostgreSQL, an administrator can end them with `pg_terminate_backend`, which also ends the application's own remaining sessions for that user.

> [!warning] Gotcha
> Changing user A's password directly in the database while `AWSCURRENT` still points to user A breaks every application that reads the current secret. Rotate once first so that applications move to user B, then change or disable user A.

## Secrets Manager vs Parameter Store

Both services can hold encrypted sensitive data and authorize access through IAM. **The requirement decides which to use**; it is too broad to say Parameter Store cannot store passwords. See also the [comparison from the Parameter Store side](ssm.md#parameter-store-vs-secrets-manager).

| Requirement                                                                                                                     | Fit                                                       |
| ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Hierarchical application configuration, public AMI parameters, or inexpensive encrypted configuration without built-in rotation | [SSM Parameter Store](ssm.md#parameter-store)             |
| Database passwords, API keys, or tokens needing secret lifecycle management                                                     | Secrets Manager                                           |
| Automatically rotate credentials and coordinate the change with the database/service                                            | Secrets Manager with the appropriate rotation integration |

> [!tip] Exam Tip
> “Automatically rotate the database password” is the strongest Secrets Manager signal. “SecureString” or “store encrypted configuration” can point to Parameter Store. Both use KMS; encryption alone does not distinguish them. Rotation may be **managed or Lambda-based**, so “rotation always requires Lambda” is incorrect.

## Pricing

The published pricing examples use **$0.40 per secret per month** and **$0.05 per 10,000 API calls**; verify the applicable Region's price when estimating a deployment. New versions created by rotation do not incur another per-secret storage charge. Lambda-based rotation and customer-managed KMS keys can add their respective service charges. Caching reduces retrieval calls, but its refresh behavior must account for rotation. See [Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/) and the [service cost notes](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html#intro_pricing).

## Example: Spring Boot + Alibaba Druid + RDS PostgreSQL Auto Rotation

Assume a Spring Boot application connects directly to [RDS PostgreSQL](rds.md) through an Alibaba Druid pool, using a dedicated application database user whose Secrets Manager secret has automatic rotation enabled. The example in this section uses the **AWS Secrets Manager JDBC Library**, which wraps the PostgreSQL JDBC driver. This is password authentication, separate from [RDS IAM database authentication](rds.md#iam-authentication).

### 1. How do new connections get the current password?

**Resolve credentials when creating a new physical database connection.** A call to `DataSource.getConnection()` usually borrows an existing session from Druid; that does not require another password check. Pool startup, growth, and replacement of closed connections do require new authentication. Reading a secret once at Spring startup and assigning its password to Druid leaves subsequent physical connections using that value unless something explicitly refreshes it. Here, we can use the aws-secretsmanager-jdbc driver to fulfill this need without self-managed refreshing logic.

Creating a new physical connection involves three steps:

1. **Druid requests a new connection** from the AWS Secrets Manager JDBC wrapper.
2. **The wrapper obtains the database username and password**, using its local cache and contacting Secrets Manager when it needs to retrieve or refresh the credentials.
3. **The wrapper passes those credentials to the PostgreSQL JDBC driver**, which authenticates and establishes the connection to RDS.

> [!warning] Gotcha
> Borrowing an existing pooled connection does not repeat credential retrieval or authentication. Once connected, the application executes SQL over the database connection; **Secrets Manager does not carry SQL traffic**.

For this **Spring Boot 3** example, add `com.alibaba:druid-spring-boot-3-starter`, `com.amazonaws.secretsmanager:aws-secretsmanager-jdbc`, and `org.postgresql:postgresql`, alongside Spring Boot's JDBC support. Select compatible released versions. The Druid starter creates and configures the data source from properties, so no custom `DataSource` bean is needed.

```properties
# application.properties — replace the endpoint and secret ID for your deployment
spring.datasource.url=jdbc-secretsmanager:postgresql://appdb.example.us-east-1.rds.amazonaws.com:5432/appdb?sslmode=verify-full&sslrootcert=/opt/certs/global-bundle.pem&connectTimeout=10
spring.datasource.driver-class-name=com.amazonaws.secretsmanager.sql.AWSSecretsManagerPostgreSQLDriver
# Now we should use Secret name or full ARN, NOT a concrete database username as usual
spring.datasource.username=prod/appdb/credentials
# Omit spring.datasource.password: the wrapper retrieves both credentials.

# Druid pool settings
spring.datasource.druid.db-type=postgresql
spring.datasource.druid.initial-size=2
spring.datasource.druid.min-idle=2
spring.datasource.druid.max-active=10
spring.datasource.druid.max-wait=10000
spring.datasource.druid.validation-query=SELECT 1
spring.datasource.druid.test-while-idle=true
```

The configuration has three parts:

- **Connection and driver:** `spring.datasource.url` and `driver-class-name` route new connections through the AWS wrapper to PostgreSQL.
- **Secret identity:** `spring.datasource.username` supplies the secret ID. The wrapper reads the actual database username and password from that secret, including when alternating-user rotation changes both.
- **Pool behavior:** `spring.datasource.druid.*` configures Druid's pool sizes, wait limit, and connection validation.

**The Druid starter enables this automatic wiring.** In the reference starter version, it binds `spring.datasource.druid.*` and uses the general `spring.datasource.*` values for connection settings not specified under the Druid prefix. It selects Druid by default when `spring.datasource.type` is absent, provided no custom `DataSource` bean exists. If the application already declares a `DataSource` bean, remove it to let the starter create the pool; merely specifying the AWS driver does not select the connection pool. See the starter's [property binding](https://github.com/alibaba/druid/blob/1.2.24/druid-spring-boot-3-starter/src/main/java/com/alibaba/druid/spring/boot3/autoconfigure/DruidDataSourceWrapper.java) and [auto-configuration conditions](https://github.com/alibaba/druid/blob/1.2.24/druid-spring-boot-3-starter/src/main/java/com/alibaba/druid/spring/boot3/autoconfigure/DruidDataSourceAutoConfigure.java).

Before running the application, configure these five prerequisites:

1. **Provide the certificates used to verify the database server.** Download the public [RDS CA certificate bundle](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html) and place it at `/opt/certs/global-bundle.pem` **inside the machine or container running Spring Boot**. The application must be able to read this file. This is an example location; if you use another path, update `sslrootcert` in the JDBC URL to match.
2. **Select the secret's AWS Region.** For a secret stored in `us-east-1`, set the application's environment variable `AWS_SECRET_JDBC_REGION=us-east-1`. This tells the AWS JDBC wrapper which regional Secrets Manager endpoint to use.
3. **Supply the database credentials in the secret.** The secret identified by `spring.datasource.username` must contain JSON fields named `username` and `password`. Keep any additional fields required by the rotation configuration, such as database connection information.
4. **Allow the application to retrieve the secret.** Its [IAM role](iam.md#iam-roles) needs `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret` for the cached-driver approach. If the secret uses a customer-managed [KMS key](kms.md), the role also needs `kms:Decrypt` authorization on that key.
5. **Provide both network paths.** The application must reach the Secrets Manager API to retrieve credentials and the RDS endpoint to open database connections. Permission to call an AWS API does not establish network connectivity.

The JDBC URL's TLS settings have separate purposes:

| Setting                                    | Meaning                                                                                                |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `sslmode=verify-full`                      | Encrypt the database connection and verify both the server certificate's trust chain and its hostname. |
| `sslrootcert=/opt/certs/global-bundle.pem` | Read the trusted CA certificates from this local file to perform that verification.                    |

The **CA bundle lets the application verify the database server's identity**. The **username/password retrieved from Secrets Manager lets the database authenticate the application**. The bundle contains public certificates, not the database password. See [PostgreSQL JDBC TLS configuration](https://jdbc.postgresql.org/documentation/ssl/).

The [AWS JDBC guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_jdbc.html) documents the Region and permissions settings. The Druid integration uses its [physical connection creation path](https://github.com/alibaba/druid/blob/master/core/src/main/java/com/alibaba/druid/pool/DruidAbstractDataSource.java).

#### Credential refresh is handled by the JDBC wrapper

When Druid needs a new physical database connection, it asks the AWS JDBC wrapper to connect. The wrapper retrieves the secret's current version—the version selected by the [`AWSCURRENT` label](#secret-versions-and-the-current-version-label)—and keeps the username/password in a local cache. Later connection attempts can reuse that cached pair without making an AWS request each time. The cache refreshes on access when its refresh time has elapsed; its configured time-to-live (TTL) defaults to one hour.

Now suppose rotation has finished: PostgreSQL accepts P2, and the current secret version also contains P2, but the wrapper's cache still contains P1. A connection attempt using P1 fails authentication. The wrapper recognizes the authentication error, such as PostgreSQL's `28P01` error code for an invalid password, refreshes its cached credentials, and retries with P2.

**When that internal retry succeeds before the caller's connection wait expires, business code receives a normal connection; the intermediate authentication exception is handled inside the wrapper and is not propagated to the application caller.** The caller may experience extra connection latency, and diagnostics may record the failed attempt, but application code does not need to catch that intermediate error, call Secrets Manager, update Druid's password, or restart the pool. See the [wrapper's connection and exception-handling code](https://github.com/aws/aws-secretsmanager-jdbc/blob/2.0.4/src/main/java/com/amazonaws/secretsmanager/sql/AWSSecretsManagerDriver.java).

#### Why a connection can still fail while rotation is in progress

The preceding recovery assumes that rotation has already published P2 as the current secret. During the [single-user rotation sequence](#how-the-single-user-strategy-works), however, the database may have switched to P2 while the `AWSCURRENT` label still points to the version containing P1. The wrapper can refresh successfully and nevertheless obtain P1 again. Its next connection attempt then fails for the same reason.

To refresh, the cache calls `DescribeSecret`, the AWS API that returns metadata including the mapping from version IDs to staging labels. It finds the version labeled `AWSCURRENT` and obtains that version's value, reusing the value if that version is already cached. If the label has not moved, another metadata read still selects the old version. **A successful refresh therefore means the AWS operation succeeded; it does not mean rotation has finished or that a newer password was returned.** See [the cache's version-selection logic](https://github.com/aws/aws-secretsmanager-caching-java/blob/2.1.1/src/main/java/com/amazonaws/secretsmanager/caching/cache/SecretCacheItem.java).

The wrapper will retry this failure too. Whether the application eventually sees an exception depends on whether those retries, or the pool's subsequent recovery, obtain a connection before the caller stops waiting.

#### Who retries, and when does the application need to act?

The exact limits below refer to **aws-secretsmanager-jdbc 2.0.4**, its declared **aws-secretsmanager-caching-java 2.1.1** dependency, and **Druid 1.2.24**. These are the source versions used to establish the behavior; check the deployed versions before relying on the same limits and defaults.

**The JDBC wrapper retries this rotation-window failure too.** It does not distinguish it from an ordinary stale-cache authentication failure. In the reference JDBC version, `MAX_RETRY=5` and the loop condition permits **six database connection attempts in total: the initial attempt plus five retries**. A recognized authentication failure triggers `refreshNow()`; exhausting the loop throws `SQLException`. A failed refresh or an unrelated error can exit earlier. See [AWSSecretsManagerDriver 2.0.4](https://github.com/aws/aws-secretsmanager-jdbc/blob/2.0.4/src/main/java/com/amazonaws/secretsmanager/sql/AWSSecretsManagerDriver.java).

Each forced cache refresh has a default random sleep of **50–100 ms**, plus API and connection latency. This is not a guarantee to wait until rotation finishes. Backoff for AWS API errors is separate: successfully reading an unchanged `AWSCURRENT` is not an API error and does not activate that exception backoff. See [SecretCacheObject 2.1.1](https://github.com/aws/aws-secretsmanager-caching-java/blob/2.1.1/src/main/java/com/amazonaws/secretsmanager/caching/cache/SecretCacheObject.java) and its [configuration defaults](https://github.com/aws/aws-secretsmanager-caching-java/blob/2.1.1/src/main/java/com/amazonaws/secretsmanager/caching/SecretCacheConfiguration.java).

**Druid provides another recovery layer for an already-running pool.** Its background connection creator catches connection SQLExceptions and can call the wrapper again. With the reference defaults `breakAfterAcquireFailure=false` and `failFast=false`, a wrapper failure does not immediately fail every waiting borrower; creation can continue, with `connectionErrorRetryAttempts` and `timeBetweenConnectErrorMillis` controlling the failure threshold and delay. A borrower can still time out while waiting for an available connection. The example sets `maxWait=10_000` for pool waiting; this is not a deadline covering every SDK call or synchronous pool initialization. See [Druid's creator and borrowing paths](https://github.com/alibaba/druid/blob/1.2.24/core/src/main/java/com/alibaba/druid/pool/DruidDataSource.java) and [defaults](https://github.com/alibaba/druid/blob/1.2.24/core/src/main/java/com/alibaba/druid/pool/DruidAbstractDataSource.java).

If the wrapper or Druid obtains a connection before the borrower's wait expires, the caller can proceed normally without receiving the earlier creation errors. If no connection becomes available in time, the pool reports an acquisition failure to the application. At that point, application policy determines whether the request should retry within its remaining time budget or return an error. Thus **automatic recovery can hide transient authentication failures, but it does not guarantee that every request succeeds during rotation**.

With `spring.datasource.druid.initial-size=2`, the starter initializes the pool synchronously. Failure to create any initial connection can propagate from `init()` and fail Spring startup; do not assume the running pool's background recovery also guarantees successful startup.

**The maintainer's job is to configure and verify timeouts/recovery, not reimplement credential refresh.** Add bounded application-level connection-acquisition retries only if needed to meet the request's availability budget after the pool's own handling. Do not blindly replay SQL or a transaction whose outcome is uncertain. To reduce the single-user mismatch window, consider [alternating-user rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html); the wrapper already refreshes the username and password together.

### 2. What happens to already-established connections during rotation?

**Changing the PostgreSQL password does not terminate sessions that already authenticated.** PostgreSQL performs password authentication during connection startup, rather than before each query. AWS also explicitly states that single-user rotation leaves open connections intact. Therefore, an in-flight [transaction](../concepts/db-transaction-model.md) can continue, and an idle physical connection in Druid can later be borrowed and used without reauthenticating. This follows the [PostgreSQL protocol lifecycle](https://www.postgresql.org/docs/current/protocol-flow.html) and [AWS rotation behavior](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html).

| Event                                                            | Effect on the application                                                                                |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Password rotates while a connection is executing SQL             | The authenticated session continues; password rotation alone does not abort its transaction.             |
| Druid lends an existing idle connection to another request       | The same database session is reused; no secret lookup or new password authentication is needed.          |
| Druid opens another connection or replaces an evicted/broken one | A new login is required; the wrapper supplies credentials and refreshes after an authentication failure. |
| A validation query succeeds on an old session                    | It proves that session works, not that the cached password can authenticate a new session.               |

For example, connections A and B authenticate with password P1. Rotation changes the database password to P2: A and B keep working. When Druid needs connection C, a cached P1 attempt fails; once P2 is published as `AWSCURRENT`, the wrapper can refresh and establish C with P2. P1 fails for new logins even while A and B remain open; its validity does not depend on how many pre-rotation connections remain. No pool restart is required to update A or B.

Connections can still close because of normal pool eviction, network failure, database failover/restart, session timeouts, or explicit session termination by custom rotation code. Those are separate events from changing the password. Alternating-user rotation likewise does **not** impose a next-rotation expiry on an already-authenticated PostgreSQL session; it changes which credentials can create new sessions.

Keep the pool running and let healthy sessions finish. Without credential refresh, replacement attempts may eventually leave the pool unable to serve requests even though rotation initially appeared harmless. With refresh, normal connection turnover can continue. Retrying connection establishment is distinct from replaying SQL or a transaction whose outcome is uncertain.

#### How the pool gradually adopts the other user

With [alternating users configured](#how-the-alternating-users-strategy-works), the same properties-based configuration handles username changes as well as password changes. Let user A be `app_user` and user B be `app_user_clone`. Initially, the wrapper caches A's credentials and Druid holds sessions authenticated as A.

When Secrets Manager rotates to B, Lambda prepares B and publishes its credentials under `AWSCURRENT`. **The wrapper receives no immediate push notification.** Because A's credentials are still valid, even new connections can temporarily continue using cached A credentials without triggering authentication-error recovery. When a later connection attempt accesses the cache after its refresh time has elapsed, a successful refresh picks up B's username/password together. Subsequent physical connections using that cache then authenticate as B. This follows the [wrapper's credential lookup](https://github.com/aws/aws-secretsmanager-jdbc/blob/2.0.4/src/main/java/com/amazonaws/secretsmanager/sql/AWSSecretsManagerDriver.java) and the cache's [refresh-on-access implementation](https://github.com/aws/aws-secretsmanager-caching-java/blob/2.1.1/src/main/java/com/amazonaws/secretsmanager/caching/cache/SecretCacheObject.java).

Existing A sessions keep their identity; they do not turn into B sessions. As Druid expands the pool or replaces closed connections, it can gradually add B sessions alongside the surviving A sessions. **No manual username/password update, pool restart, or per-rotation application intervention is required.** There is also no guaranteed time by which every session uses B: healthy A connections can remain until the pool or database closes them. Druid's lifecycle settings govern that turnover, not the JDBC secret cache.

At the next rotation, Lambda changes A's password regardless of how many A sessions remain. The earlier A password then stops working for new logins, while already-authenticated A sessions continue. If the wrapper still holds that old pair, its authentication-error refresh and retry path applies. Thus **credential rotation, cache refresh, and connection replacement are three separate processes**; maintainers configure their schedules, timeouts, and policies rather than manually coordinating each switch.

## References

- Adrian Cantrill SAA course — AWS Secrets Manager lecture; narration and architecture slide synthesized here.
- [AWS — What is Secrets Manager?](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [AWS — Secret structures and size limit](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_secret_json_structure.html)
- [AWS — Encryption and decryption](https://docs.aws.amazon.com/secretsmanager/latest/userguide/security-encryption.html)
- [AWS — Managed rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_managed.html)
- [AWS — Secret versions and staging labels](https://docs.aws.amazon.com/secretsmanager/latest/userguide/whats-in-a-secret.html)
- [AWS — Lambda rotation steps](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_lambda-functions.html)
- [AWS — Cross-Region replication](https://docs.aws.amazon.com/secretsmanager/latest/userguide/replicate-secrets.html)
- [AWS — Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/)
- [AWS — Connect to a SQL database using Secrets Manager JDBC drivers](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_jdbc.html)
- [AWS Secrets Manager JDBC Library — SDK v2-based branch and examples](https://github.com/aws/aws-secretsmanager-jdbc/tree/v2)
- [AWS — GetSecretValue API and version-stage behavior](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html)
- [AWS — Single-user and alternating-users rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html)
- [AWS — Configure database secret rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_turn-on-for-db.html)
- [AWS — PostgreSQL alternating-users rotation Lambda template](https://github.com/aws-samples/aws-secrets-manager-rotation-lambdas/blob/master/SecretsManagerRDSPostgreSQLRotationMultiUser/lambda_function.py)
- [Alibaba Druid — Physical connection creation and credential callbacks](https://github.com/alibaba/druid/blob/master/core/src/main/java/com/alibaba/druid/pool/DruidAbstractDataSource.java)
- [PostgreSQL — Protocol startup, authentication, and query lifecycle](https://www.postgresql.org/docs/current/protocol-flow.html)
