---
title: AWS Solutions Architect Associate — Exam Notes
type: certification
tags:
  - athena
  - schema-on-read
  - redshift
  - data-warehouse
  - olap
  - iam
  - vpc
  - networking
  - ec2
  - ami
  - ami-baking
  - ami-permissions
  - ami-marketplace
  - ami-lifecycle
  - block-device-mapping
  - instance-lifecycle
  - ssh
  - rdp
  - key-pair
  - iaas
  - shared-responsibility
  - host
  - tenancy
  - dedicated-host
  - dedicated-instance
  - instance-store
  - ephemeral
  - az-resilience
  - restart-vs-stop-start
  - instance-types
  - general-purpose
  - compute-optimized
  - memory-optimized
  - accelerated-computing
  - storage-optimized
  - graviton
  - burstable
  - gpu
  - fpga
  - eni
  - elastic-network-interface
  - primary-eni
  - secondary-eni
  - mac-address
  - private-dns
  - public-dns
  - split-horizon-dns
  - elastic-ip
  - eip
  - ipv6
  - source-destination-check
  - imds
  - instance-metadata
  - imdsv2
  - link-local
  - user-data
  - ssrf
  - status-checks
  - system-status-check
  - instance-status-check
  - auto-recovery
  - cloudwatch-alarm
  - sns
  - route53
  - organizations
  - cost
  - purchase-options
  - on-demand
  - spot
  - reserved-instances
  - scheduled-reserved
  - capacity-reservations
  - zonal-ri
  - regional-ri
  - savings-plans
  - byol
  - s3
  - foundations
  - object-storage
  - file-storage
  - block-storage
  - efs
  - fsx
  - ebs
  - gp2
  - gp3
  - io1
  - io2
  - block-express
  - provisioned-iops
  - per-instance-ceiling
  - iops-ratio
  - raid0
  - hdd
  - st1
  - sc1
  - throughput-optimized-hdd
  - cold-hdd
  - ebs-encryption
  - dek
  - envelope-encryption
  - aes-256
  - at-rest-encryption
  - dek-reuse
  - default-encryption
  - bucket-naming
  - bucket-limits
  - prefix
  - flat-namespace
  - regional-resilience
  - data-sovereignty
  - cloudwatch
  - multipart
  - transfer-acceleration
  - kms
  - encryption
  - sse
  - sse-kms
  - sse-s3
  - sse-c
  - cse
  - storage-classes
  - glacier
  - intelligent-tiering
  - lifecycle
  - replication
  - crr
  - srr
  - rtc
  - presigned-urls
  - s3-select
  - glacier-select
  - events
  - event-notifications
  - eventbridge
  - lambda-trigger
  - access-logs
  - audit
  - observability
  - object-lock
  - worm
  - retention
  - legal-hold
  - compliance-mode
  - governance-mode
  - immutability
  - access-points
  - mrap
  - multi-region
  - vpc-endpoint
  - failover
  - dr
  - scaling
  - horizontal-scaling
  - vertical-scaling
  - stateless
  - sticky-sessions
  - elasticity
  - load-balancer
  - containers
  - docker
  - container-image
  - container-registry
  - ecs
  - eks
  - kubernetes
  - k8s
  - control-plane
  - etcd
  - kubelet
  - pod
  - ingress
  - persistent-volume
  - node-group
  - fargate-profile
  - ecr
  - fargate
  - task-definition
  - task-role
  - cloudtrail
  - governance
  - management-events
  - data-events
  - insight-events
  - global-service-events
  - organizational-trail
  - log-file-integrity
  - event-history
  - control-tower
  - landing-zone
  - guardrails
  - account-factory
  - iam-identity-center
  - bootstrapping
  - cloud-init
  - boot-time-to-service-time
  - cloudformation
  - cfn-init
  - configsets
  - cfn-hup
  - cfn-signal
  - creation-policy
  - desired-state
  - infrastructure-as-code
  - cloudwatch-agent
  - cwagent
  - parameter-store
  - log-group
  - log-stream
  - placement-groups
  - cluster-placement-group
  - spread-placement-group
  - partition-placement-group
  - hpc
  - enhanced-networking
  - topology-aware
  - walking-the-tree
  - resolver
  - hosts-file
  - root-servers
  - nslookup
  - dig
  - dnssec
  - rrset
  - rrsig
  - dnskey
  - zsk
  - ksk
  - ds-record
  - trust-anchor
  - root-signing-ceremony
  - chain-of-trust
  - nsec3
  - cache-poisoning
  - rds
  - rds-custom
  - session-manager
  - database
  - relational
  - subnet-group
  - multi-az
  - multi-az-instance
  - multi-az-cluster
  - read-replica
  - synchronous-replication
  - semi-synchronous-replication
  - asynchronous-replication
  - quorum
  - reader-endpoint
  - cluster-endpoint
  - nvme
  - aurora
  - dms
  - database-migration
  - cdc
  - full-load
  - replication-instance
  - replication-task
  - dms-serverless
  - dcu
  - schema-conversion
  - dms-schema-conversion
  - sct
  - homogeneous-migration
  - heterogeneous-migration
  - acid
  - base
  - cap-theorem
  - consistency
  - availability
  - eventual-consistency
  - dynamodb
  - nosql
  - backup
  - snapshot
  - automated-backup
  - manual-snapshot
  - transaction-logs
  - pitr
  - point-in-time-recovery
  - restore
  - rpo
  - rto
  - cross-region-backup
  - rds-security
  - encryption-in-transit
  - encryption-at-rest
  - ssl
  - tls
  - kms-key
  - cmk
  - tde
  - transparent-data-encryption
  - cloudhsm
  - iam-authentication
  - generate-db-auth-token
  - authentication
  - authorization
  - cluster-volume
  - shared-storage
  - backtrack
  - fast-clone
  - aurora-serverless
  - acu
  - scale-to-zero
  - data-api
  - global-database
  - multi-master
  - switchover
  - database-savings-plans
  - architecture-fundamentals
  - global-components
  - regional-tiers
  - cloudfront
  - cdn
  - web-tier
  - compute-tier
  - caching-tier
  - elasticache
  - dax
  - app-services
  - decoupling
  - sqs-decoupling
  - elb
  - alb
  - nlb
  - clb
  - cross-zone-load-balancing
  - listeners
  - internet-facing
  - internal-load-balancer
  - sni
  - listener-rules
  - end-to-end-encryption
  - tls-termination
  - tcp-pass-through
  - privatelink
  - oidc
  - cognito
  - gwlb
  - launch-template
  - launch-configuration
  - auto-scaling
  - mixed-instances-policy
  - ssl-bridging
  - ssl-offload
  - session-stickiness
  - awsalb-cookie
  - sqs
  - queues
  - fifo
  - standard-queue
  - visibility-timeout
  - delay-queue
  - message-timer
  - dead-letter-queue
  - redrive-policy
  - redrive-allow-policy
  - max-receive-count
  - long-polling
  - short-polling
  - message-group-id
  - message-deduplication-id
  - fair-queues
  - fanout
  - idempotency
  - kinesis
  - data-streams
  - shards
  - partition-key
  - enhanced-fan-out
  - firehose
  - amazon-data-firehose
  - near-real-time
  - kinesis-data-analytics
  - managed-flink
  - kinesis-video-streams
  - rekognition
  - streaming
  - ingestion
  - kafka
  - msk
  - managed-kafka
  - log-vs-queue
  - replay
  - consumer-group
  - partitions
  - saa-c03
  - exam-format
  - scoring
  - practice-exams
  - skill-builder
  - study-plan
  - exam-dumps
certs:
  - aws-saa
updated: 2026-11-18
---

# AWS Solutions Architect Associate (SAA-C03)

A running collection of exam tips, gotchas, and comparison tables. For the exam format, scoring rules, and domain weights, see [Exam Format and Domains](#exam-format-and-domains); for practice exams and a study plan, see [Practice Resources and Study Plan](#practice-resources-and-study-plan). Deep-dive pages: [AppFlow](../services/appflow.md) · [Glue](../services/glue.md) · [Cognito](../services/cognito.md) · [IAM](../services/iam.md) · [VPC](../services/vpc.md) · [EC2](../services/ec2.md) · [SQS](../services/sqs.md) · [SNS](../services/sns.md) · [Kinesis](../services/kinesis.md) · [ECS](../services/ecs.md) · [ECR](../services/ecr.md) · [S3](../services/s3.md) · [RDS](../services/rds.md) · [Aurora](../services/aurora.md) · [DMS](../services/dms.md) · [Route 53](../services/route53.md) · [Organizations](../services/organizations.md) · [Control Tower](../services/control-tower.md) · [CloudWatch](../services/cloudwatch.md) · [KMS](../services/kms.md) · [EKS](../services/eks.md) · [Networking](../concepts/networking.md) · [Identity & Access](../concepts/identity-and-access.md) · [Resource vs Identity Policies](../concepts/resource-vs-identity-policies.md) · [DNS](../concepts/dns.md) · [DNSSEC](../concepts/dnssec.md) · [Database Models](../concepts/db-transaction-model.md) · [TLS/HTTPS](../concepts/tls-https.md) · [Containers](../concepts/containers.md) · [Kubernetes](../concepts/kubernetes.md) · [CloudFormation](../services/cloudformation.md) · [Cost Optimization](../patterns/cost-optimization.md) · [Scaling](../patterns/scaling.md) · [Boot Time to Service Time](../patterns/boot-time-optimization.md) · [Global & Regional Architecture](../patterns/global-regional-architecture.md).

## Exam Format and Domains

Facts in this section and in [Practice Resources and Study Plan](#practice-resources-and-study-plan) were checked against AWS pages on 2026-11-18 unless marked otherwise. Prices, question counts, and product contents change often, so recheck them before you buy.

### Format and Scoring

| Item            | SAA-C03                                                                                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Questions       | 65 in total: 50 scored and 15 unscored ([exam guide](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html)) |
| Duration        | 130 minutes ([certification page](https://aws.amazon.com/certification/certified-solutions-architect-associate/))                                                                  |
| Cost            | 150 USD, plus taxes and exchange-rate differences ([certification page](https://aws.amazon.com/certification/certified-solutions-architect-associate/))                            |
| Passing score   | 720 on a scaled score of 100–1,000 ([exam guide](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html))     |
| Validity        | 3 years ([certification page](https://aws.amazon.com/certification/certified-solutions-architect-associate/))                                                                      |
| Version history | SAA-C03 replaced SAA-C02 in August 2022                                                                                                                                            |

The exam guide lists two question types ([exam guide](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html)):

- **Multiple choice:** one correct response and three distractors.
- **Multiple response:** two or more correct responses out of five or more options. The question tells you how many to pick. "Pick 2 of 5" and "pick 3 of 6" are the common shapes, but AWS does not limit it to those two.

How scoring works, from the same exam guide:

- The 15 unscored questions are not marked, so treat every question as if it counts.
- Unanswered questions are scored as incorrect, and there is no penalty for guessing. Never leave a question blank.
- The scoring model is compensatory. You need 720 overall, not a passing score in every domain.

> [!warning] Gotcha
> The exam guide does not say whether multiple-response questions get partial credit. Study material often says "no partial credit", which is the safe assumption, but AWS does not state it in the SAA-C03 guide. Either way, aim to get every selected option right.

> [!warning] Gotcha
> As of 2026-11-18, AWS has not announced an SAA-C04 exam guide or a retirement date for SAA-C03. Pages that describe SAA-C04 as live come from practice-exam vendors and dump sites, not from AWS. Check the [AWS certification page](https://aws.amazon.com/certification/certified-solutions-architect-associate/) before believing a version change.

### Content Domains

The domain weights come from the [exam guide](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html). The "wiki coverage" column is this wiki's own mapping of where to study each domain, not AWS's task list.

| Domain                                  | Weight | Wiki coverage                                                                                                                                                                                                                                                                                                                        |
| --------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Design Secure Architectures          | 30%    | [Identity and Access](../concepts/identity-and-access.md), [IAM](../services/iam.md), [Resource vs Identity Policies](../concepts/resource-vs-identity-policies.md), [KMS](../services/kms.md), [VPC](../services/vpc.md), [WAF](../services/waf.md), [Shield](../services/shield.md), [Organizations](../services/organizations.md) |
| 2. Design Resilient Architectures       | 26%    | [Global and Regional Architecture](../patterns/global-regional-architecture.md), [Architecture Evolution](../patterns/architecture-evolution.md), [SQS](../services/sqs.md), [RDS](../services/rds.md), [Aurora](../services/aurora.md), [Route 53](../services/route53.md), [AWS Backup](../services/aws-backup.md)                 |
| 3. Design High-Performing Architectures | 24%    | [Scaling](../patterns/scaling.md), [EBS](../services/ebs.md), [ElastiCache](../services/elasticache.md), [DynamoDB](../services/dynamodb.md), [CloudFront](../services/cloudfront.md), [Kinesis](../services/kinesis.md)                                                                                                             |
| 4. Design Cost-Optimized Architectures  | 20%    | [Cost Optimization](../patterns/cost-optimization.md), [EC2 Purchase Options](../patterns/ec2-purchase-options.md), [S3 storage classes](#s3-storage-classes), [Serverless Architecture](../patterns/serverless-architecture.md)                                                                                                     |

## Architecture Fundamentals — Global vs Regional

See [Global & Regional AWS Architecture](../patterns/global-regional-architecture.md) for the full mental model, and [Resource Scope](../concepts/resource-scope.md) for per-service scope tables.

- Three architecture types: **single-region** · **single-region + DR failover** · **multi-region** (keeps working through the failure of one or more regions)
- Global components get the customer to a healthy region; three jobs with fixed service answers: **service discovery** = DNS = [Route 53](../services/route53.md); **content delivery** = CDN = CloudFront (origin can be [S3](../services/s3.md)); **health checks / failover** = [Route 53 health checks](../services/route53.md#health-checks)
- Regional applications are built from **tiers**: web ([ALB](../services/elb.md) / [API Gateway](../services/api-gateway.md) — the regional entry point that abstracts customers from the infrastructure behind it), compute ([EC2](../services/ec2.md) / Lambda / [ECS](../services/ecs.md)), caching (ElastiCache / DAX), database ([RDS](../services/rds.md) / [Aurora](../services/aurora.md) / [DynamoDB](../services/dynamodb.md) / Redshift), storage ([EBS](../services/ebs.md) / [EFS](../services/efs.md) / [S3](../services/s3.md)), app services ([Kinesis](../services/kinesis.md) / Step Functions / SQS / SNS)
- The caching tier sits **in front of** the database: consult the cache first, hit the DB only on a miss, then update the cache — "minimize reads to the database" is the trigger phrase for a caching-tier answer
- "Decouple components" is the trigger phrase for the app-services tier ([SQS](../services/sqs.md) queues)
- Core takeaway: most AWS services are **regional**; very few are truly global — classify a service by **tier** and by **global vs regional** before anything else

## Architecture Evolution — Coupling, Queues & Event-Driven

See [Architecture Evolution](../patterns/architecture-evolution.md) for the full progression (monolithic to event-driven) and the CatTube worked example.

- The whole topic is one axis: **coupling** (tight to loose), **communication** (synchronous to asynchronous), **billing** (always-on to only-while-working). Place the scenario on that axis.
- **Monolith signature** = "fails together / scales together / bills together" — any of the three phrases points at a monolith and its cost/availability downsides; it can only scale **vertically** as a unit
- **Tiered (even with load balancers) is still synchronous and coupled** — the upstream tier *waits*, so it **cannot scale to zero** and a downstream failure **cascades upstream**. Load balancers add HA + horizontal scaling, not decoupling
- **Load balancer ≠ decoupling.** A **queue** is what decouples a producer from a consumer and makes communication **asynchronous**
- **Worker fleet** = consumer [Auto Scaling Group](../services/ec2-auto-scaling.md) polling a queue, Min/Desired **0**, scales 0 to near-infinite on **queue length** (`ApproximateNumberOfMessagesVisible`), **not CPU**; producer writes the object to [S3](../services/s3.md) + a job message, then its transaction is done
- **Do not assume queues are FIFO.** [SQS](../services/sqs.md) **Standard** = best-effort ordering + at-least-once (default) → make consumers **idempotent**; **SQS FIFO** = strict ordering + exactly-once (name ends `.fifo`), the explicit choice when "ordering matters / no duplicates"
- **Event-driven** = producers emit on occurrence, a highly available **event router** (maps to **EventBridge**) delivers events to consumers via an **event bus**; nothing polls, resources are consumed **only while handling events** → strongly tied to **serverless** and cost efficiency for spiky/idle-heavy workloads

## Serverless Architecture

See [Serverless Architecture](../patterns/serverless-architecture.md) for the full PetTube worked example (S3 static site + [Cognito](../services/cognito.md) + Lambda + MediaConvert + [DynamoDB](../services/dynamodb.md)).

- **"Serverless" is not "no servers."** Servers exist; you just don't manage them. A question framing serverless as *literally* serverless is testing this.
- **Third-party token, then AWS access, means Cognito.** When users log in via Google/Facebook and then need AWS access, **Cognito** brokers the exchange and returns **temporary** credentials (the token-to-temp-credentials step is an **Identity Pool**, distinct from a User Pool's sign-in store). S3/DynamoDB cannot use the provider token as an AWS request signature. Don't pick IAM users.
- **5,000 IAM users per account** is the trigger figure that pushes consumer-facing apps toward **federated/third-party identity**. Large or unknown user counts mean **not** IAM users.
- **S3 static website + client-side JavaScript** is a fully serverless front end with **no compute**. "Download HTML/JS from S3, JS runs in the browser" is the front-end pattern.
- **S3 event, then Lambda** is the canonical event-driven trigger. "Object uploaded, then process it" points to an [S3 event notification](../services/s3.md#event-notifications) invoking Lambda (an **async** invocation).
- **Idle serverless is approximately zero cost** (consumption-based billing). Any answer implying persistent/always-on compute cost for a serverless design is wrong.
- **The managed transcoding role is MediaConvert.** **Amazon Elastic Transcoder was discontinued Nov 13, 2025**, so don't pick it as a *current* service, but recognise it if the exam still names it; learn the *role* (master file in, multiple renditions out), not the brand.
- In real designs **[API Gateway](../services/api-gateway.md)** sits between the client JS and Lambda (the web-tier front door); the diagram omits it for simplicity.

## Amazon Cognito

See [Cognito](../services/cognito.md) for the three diagram walkthroughs and [Identity and Access](../concepts/identity-and-access.md#application-tokens-vs-aws-credentials) for the token/credential distinction.

> [!tip] Exam Tip
> **User pool = sign-in + directory + application tokens. Identity pool = temporary AWS credentials through an [IAM role](../services/iam.md#iam-roles).** Use both when managed sign-in and direct AWS access are both required; either can also stand alone.

| Question trigger                                                                                                           | Answer or distinction                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Sign-up/sign-in, user directory, MFA                                                                                       | Cognito **user pool**                                                                                               |
| Mobile/web clients call [S3](../services/s3.md) or [DynamoDB](../services/dynamodb.md) directly with temporary credentials | Cognito **identity pool**                                                                                           |
| Guest AWS access without registration                                                                                      | Enable identity-pool guest access with a restricted unauthenticated role                                            |
| Logged-in users call an [API Gateway](../services/api-gateway.md) REST API                                                 | User pool + `COGNITO_USER_POOLS` authorizer                                                                         |
| Logged-in users call an API Gateway HTTP API                                                                               | JWT authorizer with the user pool issuer/audience; scopes as needed                                                 |
| Browser sign-in in front of an [ALB](../services/elb.md) application                                                       | User pool + HTTPS listener `authenticate-cognito` action                                                            |
| Many sign-in providers and direct AWS access                                                                               | User pool standardizes tokens → identity pool trusts that user pool; upstream providers still require configuration |
| Which user pool token goes into the identity pool?                                                                         | **ID token**, not access token                                                                                      |
| Avoid embedded long-term AWS keys                                                                                          | Identity pool returns temporary access key ID + secret access key + session token                                   |
| Millions of users; option proposes an IAM user per customer                                                                | Use application identities/federation; IAM has a 5,000-user account limit                                           |
| User pool JWT used directly as an S3 credential                                                                            | Incorrect: exchange for AWS credentials, or use a backend that has its own role                                     |
| Identity pool used with a `COGNITO_USER_POOLS` authorizer                                                                  | Incorrect pool type: that authorizer validates **user pool** tokens                                                 |

> [!warning] Gotcha
> An identity pool does **not** require two roles in every configuration: the unauthenticated role is for enabled guest access. It stores identity mappings, not a password directory. “Temporary credentials” still means the client holds credentials at runtime; it does not mean there are no credentials in the client.

## IAM

See [IAM](../services/iam.md) and [Identity and Access Concepts](../concepts/identity-and-access.md) for full details.

- **Deny overrides Allow**: explicit Deny in any policy wins over any Allow — across all policy sources
- Managed policies > inline policies: prefer managed for reusability and lower overhead
- IAM Users = long-term credentials; [IAM Roles](../services/iam.md#iam-roles) = temporary credentials — always use roles for [EC2](../services/ec2.md), Lambda, cross-account access
- IAM is **global** — not region-specific
- Hard limit: **5,000 IAM users per account** — for larger user bases use roles + federation
- **IAM Groups** are containers, **not identities**: no credentials, no nesting, cannot be referenced by [resource policies](../services/iam.md#policy-evaluation--priority) (only users/roles can). Limits: **10 groups per user**, default **300 groups per account**, no "all users" built-in group. Effective permissions = user policies ∪ all group policies (same Deny-wins evaluation)

### IAM Roles

See [IAM Roles](../services/iam.md#iam-roles) for full details.

- A role has **two policies**: **trust policy** (who can assume it) and **permissions policy** (what it can do)
- A principal denied by the trust policy never gets credentials — `sts:AssumeRole` fails before permissions are checked
- Assuming a role returns a **triplet**: Access Key ID + Secret Access Key + **Session Token** — the session token is the marker of temporary credentials
- Permissions changes apply to **in-flight sessions immediately** — STS does not re-issue credentials, AWS re-checks the permissions policy on every API call
- Roles can be referenced in **resource policies** (e.g. S3 bucket policy granting a role ARN) — both the role policy and the resource policy are evaluated together
- **External identities cannot directly access AWS** — Active Directory users, Google/Facebook users, and users in other AWS accounts must all assume a role first

#### When the answer is "use a role"

| Scenario                                                      | Mechanism                                                                                  |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| AWS service acting on your behalf (Lambda, EC2 → S3)          | **Service role** — trust policy trusts the service principal (e.g. `lambda.amazonaws.com`) |
| Emergency / rare elevated access                              | **Break-glass role** — assume only on demand, CloudTrail logged                            |
| Existing corporate IdP (Active Directory, Okta)               | **SAML federation** — `sts:AssumeRoleWithSAML`                                             |
| Mobile/web app with millions of users (Google/Facebook login) | **Web identity federation** — `sts:AssumeRoleWithWebIdentity`                              |
| Access to another AWS account / AWS Organizations             | **Cross-account role** — `sts:AssumeRole`                                                  |

> [!tip] Exam Tip
> Pattern-match: "company has existing AD" or "millions of mobile users" → IAM **role + federation**, not IAM users. "Lambda needs to write to S3" → **execution role**, never embedded access keys. "Two AWS accounts" → cross-account **role assumption**, not a duplicate IAM user.

> [!warning] Gotcha
> Never bake long-term IAM user access keys into Lambda code, EC2 user-data, or container images. Use a role — exam questions describing "access keys in code" are usually pointing at this anti-pattern.

## Resource Policies vs Identity Policies

See [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md) for full details.

- **Identity policy** attaches to a user/group/role; **no `Principal` field** (identity is implicit). **Resource policy** attaches to a resource (bucket, KMS key, queue, …) and **must** have a `Principal`
- **Only resource policies can grant**: (a) **cross-account** access, (b) **anonymous / public** access (`Principal: "*"`)
- **Same-account access**: Allow in **either** the identity policy **or** the resource policy is sufficient (it's a **union**, modulo Deny)
- **Cross-account access**: **both sides must Allow** (genuine **intersection**) — bucket policy names the principal *and* their home account's admin grants them the action via identity policy
- **IAM Groups cannot be principals** in resource policies — only IAM users and roles
- An IAM **role's trust policy** is a resource policy in disguise (lives on the role, names who can assume it)

> [!tip] Exam Tip
> Pattern-match by what's controlled: *"What can this user do?"* → identity policy. *"Who can access this bucket / key / queue?"* → resource policy. *"Grant anonymous read"* → resource policy (only option). *"Grant another AWS account"* → resource policy on this side + identity policy on theirs.

> [!warning] Gotcha
> Same-account behaviour is NOT an intersection — that's only true cross-account. A bucket policy in your own account that grants `s3:GetObject` to your IAM user is sufficient even if the user's identity policy says nothing about that bucket.

## S3 Foundations

See [S3 — Foundations](../services/s3.md#foundations) for full details.

- S3 is **object storage** — distinct from **file** (EFS / FSx) and **block** ([EBS](../services/ec2.md)). The access verb in the question picks the family: *mount* / *drive* / *volume* → not S3; *shared file share between EC2s* → EFS or FSx; *whole-object HTTP* / *data lake* / *land service output* → S3
- Buckets are **regional**. Classic S3 bucket names live in a **shared global namespace** across all AWS accounts and regions; newer bucket creation flows can also choose an **account-scoped regional namespace**
- Bucket name rules: **3–63 chars**, lowercase letters + digits + hyphens + periods only, must start with a letter or digit, **cannot be IP-address-shaped** (`10.0.0.1` is invalid), **no underscores**. Avoid periods for Transfer Acceleration
- **Object size: 0 B – 5 TB.** Single PUT cap is 5 GB → use [multipart upload](../services/s3.md#upload-performance) above that
- **Bucket count is per account, not per region:** current general-purpose bucket default is **10,000** per account and adjustable. Older notes that say **100 default / 1,000 via Support** are outdated. "One bucket per user" still does not scale cleanly — use **one bucket + per-user prefixes** instead
- **Objects per bucket: unlimited.** Object count is never a reason to need more buckets
- **Flat namespace.** What looks like a folder in the console is a **prefix** on the object key. There is no rename-a-directory; "moving a folder" rewrites every key
- **Regional resilience.** S3 replicates across **AZs within the home region** automatically; an AZ failure is transparent. A **region-wide failure is not** — that's what [CRR](../services/s3.md#replication) and [MRAP](../services/s3.md#multi-region-access-points-mrap--single-global-endpoint-over-a-replicated-bucket-fleet) are for
- **Data never leaves the home region** unless you explicitly configure replication. Region choice = data-sovereignty decision

### S3 foundations cheat sheet

| Keyword in the question                             | Answer                                                   |
| --------------------------------------------------- | -------------------------------------------------------- |
| "Globally unique" bucket name                       | ✅ S3 — names collide across **all accounts and regions** |
| Bucket name with `_`, uppercase, or `1.1.1.1` shape | ❌ Invalid bucket name                                    |
| Max object size                                     | **5 TB** (multipart above 5 GB)                          |
| Min object size                                     | **0 bytes**                                              |
| Max objects per bucket                              | **Unlimited**                                            |
| Default general-purpose bucket quota per account    | **10,000**                                               |
| Higher bucket quota                                 | **Adjustable** via quota increase                        |
| "Thousands of users, one bucket each"               | ❌ Use **one bucket + per-user prefix**                   |
| "Mount as a drive / volume"                         | ❌ Not S3 — use **EBS** (block) or **EFS / FSx** (file)   |
| "Network file share for Windows / Linux"            | ❌ Not S3 — use **FSx** or **EFS**                        |
| "Default destination for an AWS service's output"   | ✅ S3                                                     |
| "Large-scale distribution to many users"            | ✅ S3                                                     |
| "Offload static assets from EC2"                    | ✅ S3                                                     |
| "Folder" in S3                                      | Actually a **prefix** — flat structure underneath        |
| Data leaves the home region automatically?          | ❌ No — only via explicit **Replication** or **MRAP**     |
| Blast radius of an S3 bucket                        | **The region** it was created in                         |
| Resilience model                                    | Replicated **across AZs within the region**              |

> [!tip] Exam Tip
> When multiple storage options appear next to each other (S3, EBS, EFS, FSx), **the access verb is the discriminator**, not the data type. *"Mount", "format", "drive letter", "block device"* → not S3. *"Shared file share between EC2 instances"* → EFS (Linux) / FSx (Windows). *"Whole-object HTTP", "static asset hosting", "data lake", "service input/output"* → S3.

> [!warning] Gotcha
> The bucket limit is **per account**, not per region — moving buckets between regions does not free up headroom. If a scenario describes "needs millions of isolated stores per tenant", **bucket-per-tenant is the wrong design**: use prefixes (and optionally [access points](../services/s3.md#access-points-and-multi-region-access-points)) for tenant isolation.

## S3 Security

See [S3](../services/s3.md) for full details.

- S3 buckets are **private by default** — only the **account root** user of the owning account has initial access
- **Bucket policies** (resource policies) are how S3 implements: cross-account access, anonymous access, and conditional access — none of which identity policies can fully express
- Bucket ARNs: `arn:aws:s3:::name` (bucket itself, for list ops) vs `arn:aws:s3:::name/*` (objects). Most real policies need both
- **ACLs are legacy** — 5 permission types (READ, WRITE, READ_ACP, WRITE_ACP, FULL_CONTROL), no conditions, no Deny semantics. New buckets default to **ACLs disabled** (Object Ownership = Bucket owner enforced)
- **Block Public Access** is an **override**, not a grant: it blocks *anonymous* exposure regardless of bucket policy or ACL. Authenticated cross-account access is unaffected. New buckets get all four switches on by default
- **Explicit Deny wins** — bucket policy `Deny + Condition` is the standard way to enforce IP allow-lists or MFA-required prefixes

### Common condition patterns

| Goal                       | Condition                                                                 |
| -------------------------- | ------------------------------------------------------------------------- |
| Lock to corporate IP       | `"NotIpAddress": {"aws:SourceIp": "1.3.3.7/32"}` with `Deny`              |
| Require MFA on a prefix    | `"Null": {"aws:MultiFactorAuthAge": true}` with `Deny` (matches *no MFA*) |
| Restrict to a VPC endpoint | `"StringNotEquals": {"aws:SourceVpce": "vpce-…"}` with `Deny`             |

> [!tip] Exam Tip
> *"Bucket policy says public-read but anonymous requests get 403"* → **Block Public Access** is on. *"Want to keep accidental public-policy mistakes from leaking data"* → leave Block Public Access on at the **account** level. *"Need a public website bucket"* → must explicitly disable the relevant Block Public Access switches.

> [!warning] Gotcha
> Enabling ACLs on a new bucket is almost always the **wrong** exam answer. Modern S3 design uses bucket policies + IAM + Object Ownership (Bucket owner enforced) and skips ACLs entirely.

### S3 Pre-Signed URLs

See [S3 — Pre-Signed URLs](../services/s3.md#pre-signed-urls) for full details.

- A pre-signed URL gives one external caller **time-limited, object-scoped** access to one S3 object using the **generator's** credentials — no AWS identity needed for the caller
- Inputs to generate: bucket, key, expiry, HTTP method (**`GET`** or **`PUT`**) + generator's credentials. S3 returns a self-contained URL
- When used, S3 treats the request as if it came from the **generating identity** — same effective permissions, until expiry
- Supports **`GET`** (download) and **`PUT`** (upload). Direct browser ↔ S3 in both directions is the canonical pattern
- The bucket stays **private** — no bucket-policy or [Block Public Access](../services/s3.md#block-public-access) change required
- **Generate from an IAM user, not a role** — STS role credentials may expire before the URL does, breaking the URL even though its own expiry hasn't been reached
- **You can generate a URL for an object you don't have access to** — generation succeeds; the URL just won't work when used
- **Permissions are evaluated at use time** — current generator permissions, not at generation time. Revoking the generator invalidates outstanding URLs

#### AccessDenied diagnosis

| Symptom                                        | Cause                                                                                        |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 403 immediately, never worked                  | Generator never had permission to the object                                                 |
| Worked briefly, now 403                        | Generator was a role; **STS credentials expired** before the URL did — switch to an IAM user |
| Worked yesterday, now 403, identity unchanged  | URL has **expired** past its encoded timestamp                                               |
| Worked yesterday, now 403, URL still in window | Generator's permissions were **revoked** or modified                                         |

#### Pre-signed URL vs bucket policy

| Goal                                          | Tool                                                                     |
| --------------------------------------------- | ------------------------------------------------------------------------ |
| One external user, one object, short-lived    | **Pre-signed URL**                                                       |
| Many users, one bucket, persistent            | [Bucket policy](../services/s3.md#bucket-policies)                       |
| Public read of many objects                   | [Bucket policy](../services/s3.md#bucket-policies) with `Principal: "*"` |
| Direct browser-to-S3 upload from a web app    | **Pre-signed URL** signed for `PUT`                                      |
| Offload large media downloads from app server | **Pre-signed URL** signed for `GET`                                      |

> [!tip] Exam Tip
> Pattern-match: *"time-limited access to one object for someone without AWS creds"* → **pre-signed URL**. *"users upload directly to S3 from the browser"* → pre-signed URL signed for **`PUT`**. *"app server's bandwidth is saturated serving large media"* → pre-signed URL for **`GET`**, browser fetches directly from S3. *"pre-signed URL works briefly then 403s"* → role-as-generator + STS expiry — **use an IAM user**. *"AccessDenied on a freshly-issued URL"* → generator never had the underlying permission.

> [!warning] Gotcha
> The most common exam trap: a pre-signed URL is generated by an assumed role, works for a while, then starts returning 403 even though its own expiry hasn't elapsed. The role's STS session expired. The URL inherits the **shorter** of the two expirations. Use an IAM user for the generating identity in production patterns.

### S3 Versioning & MFA Delete

See [S3 — Versioning and MFA Delete](../services/s3.md#versioning-and-mfa-delete) for full details.

- Versioning is configured at the **bucket** level; new buckets start **Disabled**
- State machine: Disabled → Enabled → Suspended ↔ Enabled. **Enabled → Disabled is impossible** — once on, the only "off-ish" state is Suspended (existing versions are kept)
- Each write to a versioned bucket creates a new **version ID**; a `GetObject` without a `versionId` returns the current (latest) version
- `DELETE object` without a `versionId` adds a **delete marker** (soft delete — reversible by deleting the marker). `DELETE object?versionId=…` permanently removes that version (irreversible)
- **Storage cost trap**: every version is billed at full size — 5 × 5 GB = **25 GB**. Suspending versioning **does not** delete existing versions; use a lifecycle policy on noncurrent versions
- **MFA Delete** requires versioning to be enabled and gates exactly two operations: **changing the versioning state** and **permanently deleting a specific version**. Ordinary (soft) deletes and reads/writes are unaffected
- MFA Delete can only be **enabled/disabled by the bucket owner's root user**, via CLI/API (not the console). API calls must carry the MFA device serial + current code

> [!tip] Exam Tip
> Pattern-match: *"protect against accidental object deletion"* → **versioning** (sufficient on its own). *"protect against accidental **and** malicious permanent deletion, including by a compromised IAM admin"* → **versioning + MFA Delete**. *"object deleted but the team wants it back"* → if versioning was on, delete the delete marker (or `GetObject` with the previous `versionId`); if versioning was off, the data is gone.

> [!warning] Gotcha
> If a question describes "disabling versioning to reclaim storage cost", that's a trap — there's no Disable state, and Suspend doesn't drop existing versions. The right answer is lifecycle-expire noncurrent versions, or recreate the bucket.

### S3 Object Lock

See [S3 — Object Lock](../services/s3.md#object-lock) for full details.

- **Object Lock = WORM (Write-Once-Read-Many)** layered on top of [versioning](../services/s3.md#versioning-and-mfa-delete). Protects object **versions** from delete/overwrite for a defined retention period
- Can be enabled at bucket creation **or on an existing bucket** through the console, CLI, API, or SDK. Older "contact AWS Support" notes are outdated
- Enabling Object Lock **auto-enables versioning**; once enabled, neither Object Lock nor the auto-enabled versioning can be disabled — both are **one-way doors**
- **Two retention methods** that compose as an OR (either restriction blocks the action):
  - **Retention period** (days and/or years) with one of two modes — **Governance** or **Compliance**
  - **Legal Hold** — binary ON/OFF flag, no time bound
- Can be set per object version **or** as a **bucket default** (same pattern as default encryption)

#### Compliance vs Governance vs Legal Hold

| Mode                     | Time-bound?      | Bypass possible?                                                                        | Use when                                                                      |
| ------------------------ | ---------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Compliance retention** | Yes (days/years) | **No — not even by root**                                                               | Regulatory mandate to retain unmodified for N years (SEC 17a-4, HIPAA, FINRA) |
| **Governance retention** | Yes (days/years) | Yes — `s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention: true` header | Internal anti-tampering with an authorized admin escape hatch                 |
| **Legal Hold**           | No (binary flag) | Yes — `s3:PutObjectLegalHold` to toggle OFF                                             | Active legal case / indefinite preservation with no predefined end date       |

#### Required permissions

| Action                            | Permission                                                                        |
| --------------------------------- | --------------------------------------------------------------------------------- |
| Configure Object Lock on a bucket | `s3:PutBucketObjectLockConfiguration`                                             |
| Set/change retention on a version | `s3:PutObjectRetention`                                                           |
| Bypass Governance mode            | `s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention: true` header |
| Add or remove a legal hold        | `s3:PutObjectLegalHold`                                                           |

> [!tip] Exam Tip
> Pattern-match: *"WORM" / "immutable storage" / "regulatory retention"* → **Object Lock**. *"no one, including the root user, can delete or modify for N years"* → **Compliance mode** (the "even root" wording is the giveaway — MFA Delete won't satisfy this; only Compliance does). *"admins must be able to override the lock for emergencies"* → **Governance mode** + scoped `s3:BypassGovernanceRetention`. *"indefinite hold with no defined end date — active legal case"* → **Legal Hold**. *"enable Object Lock on an existing bucket"* → **console, CLI, API, or SDK**; the old Support-only answer is outdated. *"Object Lock applies to objects or to versions?"* → **versions**. *"object lock + MFA Delete"* — these solve different problems: MFA Delete gates the bucket-versioning toggle and per-version permanent delete with a second factor (compromised-IAM-admin scenario); Object Lock is the WORM/regulatory-retention answer (no-one-can-touch-this-version-period scenario).

> [!warning] Common Object Lock Gotchas
>
> - **Compliance mode is irreversible.** Misconfigured retention of 10 years means 10 years of storage cost; AWS Support cannot override it
> - **Locks are on versions, not keys.** A new PUT to the same key creates a fresh, unlocked version unless the bucket-default Object Lock config catches it
> - **Retention and legal hold are independent** — removing one does not unlock the other
> - **The bypass header is default in the AWS Console** — console users with `s3:BypassGovernanceRetention` can override Governance locks without manually adding any header. Scope this permission tightly
> - **Versioning cannot be suspended** on an Object-Lock-enabled bucket — versioning is permanent for the life of the bucket

### S3 Upload Performance — Multipart + Transfer Acceleration

See [S3 — Upload Performance](../services/s3.md#upload-performance) for full details.

- **Single `PutObject` is capped at 5 GB** — anything larger **requires** multipart upload (not optional)
- AWS recommends multipart for objects **≥ 100 MB**; the SDK / CLI transfer manager auto-picks multipart above ~8 MB, so this is usually handled for you
- Multipart parts: **5 MiB–5 GiB each** (last part can be smaller), **max 10,000 parts** → max object size ~5 TB
- Failed multipart parts **retry individually** — single PUT failures restart from byte 0
- **Incomplete multipart uploads still incur storage costs.** Standard fix: lifecycle rule with **`AbortIncompleteMultipartUpload`** after N days
- **Transfer Acceleration** uses **CloudFront edge locations** + the **AWS global backbone** to bypass the public-internet long-haul. Bucket-level opt-in (off by default), separate endpoint `<bucket>.s3-accelerate.amazonaws.com`
- TA bucket-name requirement: **DNS-compatible, no periods** (the dotted hostname would break TLS hostname matching)
- TA costs **extra per GB** transferred; AWS provides a Speed Comparison tool to verify benefit before enabling
- Multipart and TA are **orthogonal and combinable** — multipart parallelizes the bytes, TA shortens the network path

| Goal                                                 | Mechanism                                    |
| ---------------------------------------------------- | -------------------------------------------- |
| Upload a 50 GB file (reliability, throughput)        | **Multipart upload** (required — > 5 GB)     |
| Globally distributed clients uploading to one bucket | **Transfer Acceleration**                    |
| Large files **and** geographically far clients       | **Both** — multipart over the TA endpoint    |
| Bulk transfer of petabytes from on-prem              | **Snowball / DataSync** (beyond TA's domain) |

> [!tip] Exam Tip
> Pattern-match: *"large object, unreliable connection, want to resume on failure"* → **multipart upload**. *"users on multiple continents uploading to one bucket"* → **Transfer Acceleration**. *"file > 5 GB"* → multipart is **required**, not a choice. *"reduce orphaned-upload storage charges"* → lifecycle rule with `AbortIncompleteMultipartUpload`.

> [!warning] Gotcha
> Transfer Acceleration cannot be enabled on a bucket whose name contains **a period** (`.`). If a question mentions a bucket like `my.company.data` and asks why TA isn't working, the answer is **the bucket name must be DNS-compatible and dotless**.

### S3 Storage Classes

See [S3 — Storage Classes](../services/s3.md#storage-classes) for full details.

- **Default to S3 Standard** — only move off it when the question explicitly mentions infrequent access, archival, or a replaceable/regenerable copy
- Every class **except One Zone-IA** is designed for **11 nines durability** across **≥ 3 AZs**. One Zone-IA's 11 nines is **conditional on the single AZ not failing**
- **Durability ≠ availability.** Standard = 11 nines durability + 4 nines (99.99%) availability. Standard-IA availability drops to 99.9%; One Zone-IA to 99.5%
- **S3 Standard is the only class with no retrieval fee, no minimum size, and no minimum duration** — every other class adds at least one
- **Public access requires hot tiers**: Standard, IA, One Zone-IA, GIR, accessible IT tiers. Glacier Flexible and Glacier Deep Archive **cannot be made public** — there's no way to serve an object when reading it requires a multi-hour job
- **Restoring a Glacier object** stages a temporary copy in Standard-IA storage for the requested availability window — the **original Glacier object isn't moved**. Costs: retrieval fee + temporary IA storage. To permanently un-Glacier, change class via `CopyObject` after restore

#### Class-by-class anchor table

| Class                          | First-byte latency | Min. duration              | Min. size | Retrieval fee                               | Key trade-off                                                   |
| ------------------------------ | ------------------ | -------------------------- | --------- | ------------------------------------------- | --------------------------------------------------------------- |
| **Standard**                   | ms                 | None                       | None      | None                                        | The default — baseline cost                                     |
| **Standard-IA**                | ms                 | 30 days                    | 128 KB    | $0.01/GB                                    | ~½× storage cost; retrieval + minimums                          |
| **One Zone-IA**                | ms                 | 30 days                    | 128 KB    | $0.01/GB                                    | ~20% cheaper than IA; **single AZ**                             |
| **Glacier Instant Retrieval**  | ms                 | 90 days                    | 128 KB    | $0.03/GB                                    | Cheap storage + instant access; per-GET fee                     |
| **Glacier Flexible Retrieval** | minutes–hours      | 90 days                    | 40 KB     | Expedited $0.03 / Std $0.01 / **Bulk free** | Archival; retrieval job required                                |
| **Glacier Deep Archive**       | hours              | 180 days                   | 40 KB     | Std $0.02 / Bulk $0.0025                    | Cheapest tier; **no Expedited** option                          |
| **Intelligent-Tiering**        | varies by tier     | None (90d archive-instant) | None      | None (monitoring fee)                       | Auto-moves objects; useful only for **unknown** access patterns |

#### Glacier retrieval-tier timing

| Tier                         | Restore time | Per-GB fee |
| ---------------------------- | ------------ | ---------- |
| Glacier Flexible — Expedited | 1–5 min      | $0.03/GB   |
| Glacier Flexible — Standard  | 3–5 hr       | $0.01/GB   |
| Glacier Flexible — Bulk      | 5–12 hr      | **Free**   |
| Deep Archive — Standard      | within 12 hr | $0.02/GB   |
| Deep Archive — Bulk          | up to 48 hr  | $0.0025/GB |

#### Lifecycle Configuration

See [S3 — Lifecycle Configuration](../services/s3.md#lifecycle-configuration) for full details.

- **Two action types**: **Transition** (change storage class) and **Expiration** (delete object or version). Rule scope: whole bucket, prefix filter, tag filter, object size filter, or — on versioned buckets — object versions
- **Lifecycle is time-based, not access-based.** *"Move when frequently/rarely accessed"* → that's **Intelligent-Tiering**, not lifecycle
- **Transitions are one-way down the cost ladder** — to move *back up* to a warmer class, use `CopyObject`, not a lifecycle rule
- **30-day Standard minimum** between upload and any lifecycle transition to Standard-IA or One Zone-IA (direct PUT into IA at upload time is fine; only the lifecycle path is gated). **Same rule applies to noncurrent versions** — they must be at least 30 days noncurrent before transitioning to Standard-IA / One Zone-IA
- **Single-rule minimum-duration gap**: within one rule, consecutive transitions must respect the **destination class's minimum storage duration**. The IA-to-Glacier case (30-day IA minimum) is the most common, but the rule generalizes — e.g. a rule that says GIR @ 4 days then Deep Archive @ 20 days is rejected (GIR's 90-day minimum means Deep Archive can't fire before day 94). **Workaround: use two separate rules** — the constraint doesn't apply across rules (you still pay the destination's minimum-duration charges)
- **Destination-class minimums apply** — a lifecycle move into Standard-IA still triggers IA's 30-day minimum charge
- **Objects < 128 KB don't transition by default** (since September 2024). Override with an `ObjectSizeGreaterThan` filter on the rule, or set the `x-amz-transition-default-minimum-object-size` header on `PutBucketLifecycleConfiguration`. Pre-Sept-2024 configurations retain the old behavior until edited
- **Pending / Failed replication objects are skipped** by lifecycle on versioning-enabled / versioning-suspended buckets — fix replication before expecting transitions to fire
- **Lifecycle is the same engine** behind `AbortIncompleteMultipartUpload`, noncurrent-version expiration, and storage-class transitions — different actions, same machinery

##### The transition waterfall

Every class can transition down to a colder class, with **one exception**: **One Zone-IA cannot transition to Glacier Instant Retrieval**. Every other source class (Standard, Standard-IA, Intelligent-Tiering, even Glacier Instant Retrieval itself) can target GIR. Memorize this single asymmetry.

| From                       | Can transition to                                                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Standard                   | All other classes                                                                                                                                                                                                                                                  |
| Standard-IA                | Intelligent-Tiering, One Zone-IA, GIR, Glacier Flexible, Glacier Deep Archive                                                                                                                                                                                      |
| Intelligent-Tiering        | Varies by IT access tier — narrows as the source tier gets colder (Frequent/Infrequent → One Zone-IA + Glacier classes; Archive Instant Access → GIR + Glacier classes only; Archive Access → Glacier Flexible/Deep only; Deep Archive Access → Deep Archive only) |
| One Zone-IA                | Glacier Flexible, Glacier Deep Archive ⚠️ (**not** GIR)                                                                                                                                                                                                             |
| Glacier Instant Retrieval  | Glacier Flexible, Glacier Deep Archive                                                                                                                                                                                                                             |
| Glacier Flexible Retrieval | Glacier Deep Archive                                                                                                                                                                                                                                               |
| Glacier Deep Archive       | (terminal)                                                                                                                                                                                                                                                         |

> [!tip] Exam Tip
> Pattern-match: *"transition objects to a cheaper tier automatically as they age"* → **lifecycle transition**. *"delete objects after N days"* → **lifecycle expiration**. *"move data based on access pattern"* → **Intelligent-Tiering**, not lifecycle. *"Standard → Glacier on day 5"* → blocked by the **30-day Standard minimum**. *"single rule: GIR day 4 → Deep Archive day 20"* → blocked by the **single-rule minimum-duration gap** (GIR's 90-day minimum); the Deep Archive transition must be on day 94 or later, or split into two rules. *"One Zone-IA → Glacier Instant Retrieval"* → **not allowed**; route via a hot tier with `CopyObject` first. *"my lifecycle rule isn't moving any of my small log files"* → since Sept 2024, **objects < 128 KB don't transition by default** — add `ObjectSizeGreaterThan`.

> [!warning] Gotcha — small objects don't transition by default, and archive overhead makes them worse
> Two compounding factors make small-object archival a trap: **(1)** since September 2024, lifecycle skips objects < 128 KB by default unless you explicitly override the size filter, and **(2)** every object archived to Glacier Flexible / Deep Archive carries **8 KB at Standard rates + 32 KB at Glacier rates** of fixed metadata overhead. For tiny files the overhead alone can exceed the object itself — aggregate into larger objects (tarballs, columnar blobs) before archiving.

> [!tip] Exam Tip
> Pattern-match by **access frequency + retrieval tolerance**:
>
> - *"frequently accessed, important"* → **Standard**
> - *"accessed monthly, important"* → **Standard-IA**
> - *"replica / regenerable / derived data"* → **One Zone-IA**
> - *"accessed quarterly but must be instant"* → **Glacier Instant Retrieval**
> - *"archived, hours OK to retrieve"* → **Glacier Flexible Retrieval**
> - *"compliance archive, up-to-48 hours OK"* → **Glacier Deep Archive**
> - *"access pattern unknown or shifting over time"* → **Intelligent-Tiering** (and *only* then — for static patterns, the monitoring fee eats the savings)

> [!warning] Gotcha — "Glacier" doesn't mean public
> Glacier Flexible Retrieval and Glacier Deep Archive **cannot be made publicly accessible** and don't work with static website hosting. If a question describes *"archival storage that must remain publicly downloadable"*, the answer is **Glacier Instant Retrieval**, not Flexible or Deep.

> [!warning] Gotcha — One Zone-IA's 11 nines is conditional
> Marketing says 11 nines across all classes. For One Zone-IA, that math **assumes the single AZ doesn't fail** — an AZ-level loss event takes the data with it. Never use One Zone-IA for the only copy of critical data.

> [!warning] Gotcha — Intelligent-Tiering isn't free
> IT replaces retrieval fees with a **monitoring/automation fee per 1,000 objects**. For static, predictable access patterns, IT costs *more* than the right manual class. The right answer is IT only when the prompt explicitly says access patterns are **unknown or changing**.

### S3 Replication

See [S3 — Replication](../services/s3.md#replication) for full details.

- **CRR** (Cross-Region) vs **SRR** (Same-Region) differ only in *where* the destination lives. Both work **same-account** or **cross-account**
- Replication is configured on the **source bucket** + an IAM role (S3 service trust policy; permissions on source `Get*` + destination `Replicate*`)
- **Same-account**: IAM role is enough. **Cross-account**: also need a **bucket policy on the destination** allowing the source-account role
- **Versioning is required on both buckets** — non-negotiable
- **Not retroactive** — only new objects from configuration time onward. To backfill existing objects, use **S3 Batch Replication** (a separate, explicit job)
- **One-way by default** — bidirectional requires two configurations (one on each bucket)
- **Replication Time Control (RTC)**: adds a **15-minute SLA** + per-object replication metrics. *"15 minutes"* → **RTC**
- **Object ownership**: by default replicated objects stay owned by the **source account** — in cross-account setups, **override** so the destination account owns them, otherwise the destination's IAM users can't read their own bucket
- **Storage class**: same as source by default; can be overridden (often **One Zone-IA** for the secondary copy)

#### What is NOT replicated

| Not replicated                                          | Why / detail                                                                                                                                   |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre-existing objects**                                | Configure **S3 Batch Replication** to backfill                                                                                                 |
| **Lifecycle transitions / expirations**                 | System events aren't replicated; user-driven events are. Configure the same lifecycle rule on **both** buckets if you want symmetric behaviour |
| **Glacier Flexible Retrieval and Deep Archive objects** | Treat those classes as a separate product. (Glacier *Instant* Retrieval **is** replicable)                                                     |
| **Delete markers** (by default)                         | Can be opted in per rule                                                                                                                       |
| **Versioned (permanent) deletes**                       | **Never** replicated, by design — replication is for survival, not synchronized destruction                                                    |
| **Objects not owned by the source account**             | Replication requires source-account ownership. Cross-account writes with third-party ownership are silently skipped                            |

#### Encryption support

| Method      | Replication                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------------- |
| Unencrypted | ✅ Default                                                                                                  |
| **SSE-S3**  | ✅ Default                                                                                                  |
| **SSE-KMS** | ✅ Requires extra config — KMS key permissions on both sides, destination key named in the replication rule |
| **SSE-C**   | ⚠️ Only recently supported — historically blocked                                                           |

#### Use-case lookup

| Scenario                                                                        | CRR or SRR?           |
| ------------------------------------------------------------------------------- | --------------------- |
| DR / survive region failure                                                     | **CRR**               |
| Read latency reduction for global users                                         | **CRR**               |
| Centralize logs from many buckets                                               | **SRR**               |
| Prod ↔ test data sync                                                           | **SRR**               |
| Sovereignty-bound resilience (data must stay in region, isolated audit account) | **SRR** cross-account |

> [!tip] Exam Tip
> Pattern-match: *"replicate to another region for DR"* → **CRR**. *"15-minute SLA"* → **RTC**. *"backfill existing objects after enabling replication"* → **S3 Batch Replication**. *"destination account can't read replicated objects in its own bucket"* → **object-ownership override** wasn't set on the rule. *"replication doesn't propagate deletes"* → **default behaviour** — delete markers can be opted in; permanent versioned deletes never replicate. *"lifecycle expirations don't propagate"* → by design — system events aren't replicated.

> [!warning] Gotcha — Versioning prerequisite vs the one-way door
> Replication requires versioning on both buckets. Once enabled, [versioning can only be Suspended, never Disabled](../services/s3.md#state-machine--one-way-door). Turning on replication is therefore a partial one-way door for the destination bucket — you can stop replication later, but the destination is permanently a versioned bucket.

> [!warning] Gotcha — Cross-account ownership
> The most common cross-account exam trap: replication succeeds, the bytes arrive in the destination, but the destination account's IAM users get **403** trying to read them — because the source account still owns the objects. The fix is the **"Change object ownership to destination bucket owner"** option in the replication rule, set at configuration time.

### S3 Select & Glacier Select

See [S3 — Server-Side Filtering](../services/s3.md#server-side-filtering--s3-select--glacier-select) for full details.

- **S3 Select** and **Glacier Select** push a SQL-like filter **into the storage service** so the application only retrieves the rows/columns it actually needs — not the whole object
- **S3 Select is no longer available to new customers**. Keep the concept for legacy/exam recognition and existing customers; do not treat it as a first-choice feature for new designs
- The architectural win: filtering moves **from the application to S3**, so the network only carries the bytes the app uses. Up to **400% faster**, up to **80% cheaper** on data transfer
- **Supported formats**: **CSV**, **JSON**, **Parquet** (S3 Select); **BZIP2** compression for CSV and JSON
- **One object at a time** — no joins, no aggregations across objects. Multi-object queries belong on **[Athena](../services/athena.md)**, not S3 Select
- **S3 Select** = synchronous, against any hot tier (Standard / IA / One Zone-IA / GIR), millisecond latency
- **Glacier Select** = a **retrieval job** against Glacier Flexible / Deep Archive — inherits the source class's [retrieval tier](../services/s3.md#retrieval-tiers) (Expedited / Standard / Bulk). Not an instant query

#### Decision lookup

| Scenario                                                                   | Answer                                        |
| -------------------------------------------------------------------------- | --------------------------------------------- |
| "Retrieve only specific rows/columns from a large CSV/JSON/Parquet object" | **S3 Select**                                 |
| Same scenario but the object is in Glacier Flexible / Deep Archive         | **Glacier Select** (still a retrieval job)    |
| "Query across many S3 objects, with joins or aggregations"                 | **Athena**, not S3 Select                     |
| "App downloads the whole object then filters in memory"                    | The **anti-pattern** S3 Select fixes          |
| Source object is an image / video / binary archive                         | **Plain GetObject** — S3 Select doesn't apply |

> [!tip] Exam Tip
> Pattern-match: *"reduce data transferred from S3 / filter at the source"* → **S3 Select**. *"SQL-like expression on one object"* → **S3 Select**. *"400% faster / 80% cheaper"* → S3 Select / Glacier Select. *"object lives in Glacier and we need to filter on restore"* → **Glacier Select** (still a retrieval job — Expedited / Standard / Bulk choice still applies). *"query across thousands of files / joins / aggregations"* → **Athena**.

> [!warning] Gotcha — S3 Select is not a query engine
> The exam likes to dangle S3 Select as a tempting answer to multi-object query scenarios. Don't bite. **One object per call** is the hard boundary. Joins, aggregations across objects, schema catalog → **Athena**. Format outside CSV / JSON / Parquet → **not S3 Select** either.

### S3 Event Notifications

See [S3 — Event Notifications](../services/s3.md#event-notifications) for full details.

- S3 emits notifications when objects are **created, deleted, restored (from Glacier), or replicated** — the trigger for event-driven workflows around the bucket
- **Three native destinations only**: **Lambda**, **SQS**, **SNS** — nothing else. (EventBridge is a separate, broader integration.)
- The destination needs a **resource policy** trusting the **`s3.amazonaws.com`** service principal — Lambda function policy / SQS queue policy / SNS topic policy. Without it, **delivery silently fails**
- Events are delivered as **JSON**; Lambda receives it inside the `event` argument
- **Object Created** fires on `PUT`, `POST`, `COPY`, **completed multipart upload**
- **Object Removed** fires on standard deletes + **delete-marker creation** (on versioned buckets)
- **Object Restore** fires when Glacier restore **starts** AND **completes**
- **Object Replication** fires on **missed 15-min RTC SLA**, replicated-after-threshold, no-longer-tracked, and **failed** replication
- **Filtering**: native event notifications support only **prefix/suffix** on the object key — anything richer needs EventBridge

#### Native S3 Events vs EventBridge

| Need                                                                                                       | Pick                                                               |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Trigger Lambda on object upload, simple bucket-local config                                                | **S3 Event Notifications** (or EventBridge)                        |
| Destinations beyond Lambda/SQS/SNS (Step Functions, [Kinesis](../services/kinesis.md), custom buses, SaaS) | **EventBridge**                                                    |
| Rich event-pattern filtering (any field, not just key prefix/suffix)                                       | **EventBridge**                                                    |
| Lifecycle / intelligent-tiering / fine-grained replication events                                          | **EventBridge**                                                    |
| Cross-account fan-out                                                                                      | **EventBridge**                                                    |
| Default for new workflows                                                                                  | **EventBridge** unless a specific reason to use the native feature |

> [!tip] Exam Tip
> Pattern-match: *"trigger Lambda when a file lands in S3"* → **S3 Event Notifications → Lambda** (also acceptable: EventBridge → Lambda). *"three destinations / Lambda, SQS, SNS only"* → native S3 Event Notifications. *"events delivered as JSON"* → yes. *"what permits S3 to invoke / send / publish"* → **resource policy on the destination** trusting **`s3.amazonaws.com`**. *"notify when Glacier restore completes"* → `s3:ObjectRestore:Completed`. *"detect failed cross-region replication"* → `s3:ObjectReplication:*` (or EventBridge). *"events not arriving, no error in CloudTrail on the bucket side"* → check the **destination's resource policy** — bucket isn't the requester, so IAM errors surface on the destination, not the bucket.

> [!warning] Gotcha — silent failure mode
> S3 Events fail silently if the destination's resource policy is missing or wrong. The S3 console wires up Lambda permissions for you when you add the destination, but if you build the configuration via CloudFormation / Terraform / CLI, the function/queue/topic policy is a **separate resource you must declare**. EventBridge has the same constraint on its targets.

### S3 Access Logging

See [S3 — Access Logging](../services/s3.md#access-logging) for full details.

- Async, best-effort request-level audit log of every operation against a **source bucket**, delivered as **Apache-style** text files into a **target bucket** (same region and same account)
- Configured on the **source bucket** via the console or **`PUT Bucket logging`** API; the XML config names the target bucket and an optional `TargetPrefix`
- Log delivery is performed by the AWS-managed S3 logging delivery path — *not* the source-bucket owner or the requester
- The target bucket grants write access to the **`logging.s3.amazonaws.com`** service principal. Use a **bucket policy** for modern buckets; an ACL grant to the legacy S3 Log Delivery Group is possible only when ACLs are enabled and is not recommended
- **Best-efforts, hours of latency** — enabling logging or changing config can take a few hours; delivery is not guaranteed real-time. Don't build alerting on top of it
- **Format**: log files contain many records; **records are newline-delimited**, **attributes within a record are space-delimited**. Conceptually identical to an Apache access log
- **One target bucket, many sources** — use distinct `TargetPrefix` values per source bucket
- **No built-in retention** — attach a **Lifecycle rule** to the target bucket (transition to Glacier, expire after retention window) or logs accumulate forever

#### Access Logs vs CloudTrail Data Events

| Need                                                                                   | Pick                                      |
| -------------------------------------------------------------------------------------- | ----------------------------------------- |
| Bulk, cheap, request-level audit log; no per-event charge                              | **S3 Access Logs**                        |
| Near-real-time, structured JSON, full IAM context (principal, session, MFA, source IP) | **CloudTrail data events**                |
| Native integration with CloudWatch / EventBridge / Detective for alerting              | **CloudTrail data events**                |
| Compliance evidence of every read/write at minimal cost                                | **S3 Access Logs** + lifecycle to Glacier |

> [!tip] Exam Tip
> Pattern-match: *"who accessed which objects in my bucket, cheapest option"* → **S3 Access Logs**. *"where do I enable logging / where do logs go"* → enable on **source bucket**, delivered to **target bucket**. *"target bucket constraints"* → **same region and same account**, destination bucket must not have Object Lock enabled, no Requester Pays. *"how does S3 get permission to write into the target bucket"* → **bucket policy** granting `s3:PutObject` to **`logging.s3.amazonaws.com`**. *"why is logging not showing up immediately"* → **best-efforts; can take hours**. *"record / attribute delimiter"* → **newline / space**. *"log retention"* → **your responsibility — use Lifecycle**. *"need real-time / structured / IAM-context-rich logs for security automation"* → **CloudTrail data events**, not access logs. *"need to trigger Lambda on object upload"* → **Event Notifications**, not access logs (access logs are batch/delayed).

> [!warning] Gotcha — ACLs disabled changes the answer
> Modern S3 defaults to ACLs disabled (Object Ownership = Bucket owner enforced). In that mode, an ACL grant to the S3 Log Delivery Group cannot work. The fix is a target-bucket policy granting `s3:PutObject` to `logging.s3.amazonaws.com`, not re-enabling ACLs.

### S3 Access Points & Multi-Region Access Points

See [S3 — Access Points and MRAP](../services/s3.md#access-points-and-multi-region-access-points) for full details.

- **Regular access points** are named DNS endpoints attached to a **single** bucket — each with its own resource policy (≤ 20 KB), network origin (`Internet` or `VPC`), and BPA settings. Solve the "one bucket policy is too small / too complex for many teams" problem
- **Multi-Region Access Points (MRAP)** are a **single global endpoint** over **2+ buckets in different regions**, with latency-based routing — solve the "single endpoint for multi-region S3" and "automatic regional failover" problems
- Both are **naming/routing layers, not permissions systems** — like a DNS CNAME with a policy attached. The underlying **bucket policy must still allow** the request
- Both live in the **`aws s3control`** CLI namespace (not `aws s3` / `aws s3api`)
- Both have **account-unique** names (not globally unique like bucket names)
- **Two-layer permissions rule**: matching permissions in both policies, **or** delegation pattern (recommended) — bucket policy uses `s3:DataAccessPointAccount` to allow any AP in the account; AP policies hold the real granularity
- AP-specific condition keys: `s3:DataAccessPointArn`, `s3:DataAccessPointAccount`, `s3:AccessPointNetworkOrigin` (`Internet` or `VPC`)

#### Regular Access Points — quick recall

- **Network origin** (`Internet` / `VPC`) and **BPA settings** are **immutable** after creation
- **Bucket association** is immutable — recreate the AP to change it
- AP must be in the **same region as the bucket**; regional ARN format `arn:aws:s3:<region>:<account>:accesspoint/<name>`
- **VPC-origin pattern** requires three pieces to line up: VPC-origin AP, VPC endpoint policy naming **both** the bucket ARN **and** the AP ARN, and (typically) a bucket policy denying any request whose `s3:AccessPointNetworkOrigin != VPC`
- **SCP guardrail** for org-wide VPC-only enforcement: `Deny s3:CreateAccessPoint` when `s3:AccessPointNetworkOrigin != VPC`
- **Cannot be a [Replication](../services/s3.md#replication) destination** — beware when locking a bucket to VPC-only APs (replication writes directly to the bucket, not through an AP)
- **Limit**: 10,000 access points per account per region (soft); 20 KB AP policy size

#### MRAP — quick recall

- Backing buckets are **fixed at creation** — cannot be added or removed later; recreate the MRAP to change
- **Versioning** must be enabled on every backing bucket (CRR requires it)
- **Replication is separate** — configure mesh **CRR** between the backing buckets, typically via the "Replicate among all specified buckets" template
- ARN format `arn:aws:s3::<account>:accesspoint/<alias>` — note the **empty region segment** (regionless)
- **Internet origin only** — MRAP does **not** support VPC origin (regular APs do)
- **Failover mode**: `active–active` (default; any active bucket can be picked) or `active–passive` (passives only used when no actives are healthy) — the active/passive variant is the **DR** pattern
- **Creation latency** — typically 10–30 minutes, AWS quotes up to **24 hours** worst-case
- **Consistency lag** — MRAP routing is regional/latency-based, async CRR replicates *after* the write. A read from a different region than the write *can* return `NoSuchKey` before replication finishes. **Mitigate** with [Replication Time Control (RTC)](../services/s3.md#replication-time-control-rtc) (99.9% within 15 min) or read-from-write-region in the app
- **Cost** — per-GB data routing fee on top of S3 storage, CRR transfer, and (optional) RTC

#### Access Points & MRAP — decision lookup

| Keyword in question                                                    | Answer                                                                                             |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| "Bucket policy too large / too complex / 20 KB limit"                  | **Regular S3 Access Points**                                                                       |
| "Many teams need scoped views into one bucket"                         | **Regular Access Points** + **delegation** (`s3:DataAccessPointAccount`)                           |
| "Restrict bucket access to one specific VPC"                           | **VPC-origin Access Point** + VPC endpoint policy naming both ARNs                                 |
| "Single global S3 endpoint" / "lowest-latency S3 bucket from anywhere" | **MRAP**                                                                                           |
| "Active/active S3 in multiple regions behind one endpoint"             | **MRAP** + **mesh CRR**                                                                            |
| "Automatic regional failover for S3"                                   | **MRAP** with **active/passive** failover configuration                                            |
| "Add another bucket to the MRAP later"                                 | **Not supported** — backing buckets fixed at creation; recreate the MRAP                           |
| "Object not found right after writing through the MRAP"                | **Consistency lag** — async CRR hasn't propagated; mitigate with **RTC** or read-from-write-region |
| "Guaranteed 15-minute replication SLA"                                 | **[Replication Time Control (RTC)](../services/s3.md#replication-time-control-rtc)**               |
| "MRAP / AP name uniqueness"                                            | **Account-scoped** (bucket names are globally unique; APs and MRAPs are not)                       |
| "CLI to use an MRAP"                                                   | `s3://<MRAP-ARN>/key` — **ARN**, not bucket name                                                   |
| "Access point cannot be a…"                                            | **[Replication](../services/s3.md#replication) destination**                                       |
| "SCP to force VPC-only access points"                                  | `Deny s3:CreateAccessPoint` when `s3:AccessPointNetworkOrigin != VPC`                              |
| "Bucket policy delegation to access points"                            | Use `s3:DataAccessPointAccount` to grant `*` to any AP in the account                              |

> [!tip] Exam Tip
> Pattern-match: *"bucket policy too large / many teams"* → **regular Access Points** with delegation. *"VPC-only S3"* → **VPC-origin AP** + VPC endpoint policy + (optionally) deny-non-VPC bucket policy. *"single global S3 endpoint / latency-route to closest bucket / multi-region S3 DR"* → **MRAP** + **mesh CRR** (+ **RTC** if the question stresses consistency or 15-minute SLA). *"object not found right after writing through MRAP"* → **consistency lag**; RTC narrows but doesn't eliminate it. *"can I add a bucket to the MRAP later?"* → **No** — fixed at creation. *"who grants the permission in a cross-account access point?"* → the **bucket owner** (the AP is just a doorway; the bucket policy is the real authority).

> [!warning] Common Gotchas
>
> - **Access points are not a permissions system.** The bucket policy is still the final authority. Forgetting this is the most common AP exam trap.
> - **Access points cannot be replication destinations.** A "VPC-only access points" bucket policy will silently break [Replication](../services/s3.md#replication) unless the replication role is allow-listed as an exception.
> - **MRAP gives up strong read-after-write consistency.** Regular S3 has it; MRAP does not. RTC narrows the window; only same-region read-from-write eliminates it.
> - **MRAP backing buckets are fixed.** Plan the bucket set up front — there's no add/remove API.
> - **MRAP has no VPC origin.** Only regular APs do. If you need both VPC scoping and multi-region routing, you must layer it at the application level.
> - **CLI namespace is `aws s3control`** for both AP types — not `aws s3` or `aws s3api`. Console wizards hide this distinction.

## VPC & Networking

See [VPC](../services/vpc.md) and [Networking Concepts](../concepts/networking.md) for full details.

### Firewall Rules

- Both [NACL](../services/vpc.md#network-acl-nacl) and [Security Group](../services/vpc.md#security-groups-sg) operate at **Layer 3/4** — not Layer 7; they filter by IP, protocol, and port only
- ICMP rules = Layer 3 only (no ports); TCP/UDP rules = Layer 4
- Layer 7 (HTTP content, headers, URLs) is handled by **[WAF](../services/waf.md)**, not NACL or SG; a firewall only understands its own layer and below (see [Layer 7 Firewalls](../concepts/networking.md#layer-7-application-firewalls))
- **NACL** = stateless, attaches to **subnets** (L3 constructs), supports explicit Deny → use to block bad IPs
- **Security Group** = stateful, attaches to **ENIs** (L3/4 constructs), no explicit Deny → use for fine-grained per-resource control
- Because NACLs are stateless: always allow **ephemeral ports (1024–65535)** for return traffic
- The real distinction between NACL and SG is **stateful vs stateless** + **attachment point**, not OSI layer

### VPC Peering

See [VPC Peering](../services/vpc-peering.md) for the topology diagram, reciprocal routes, and DNS configuration.

> [!tip] Exam Tip
>
> - **Exactly two VPCs per connection**, within or across accounts/Regions; CIDRs must not overlap, including secondary blocks.
> - **A–B + B–C does not provide A–C.** Add a direct A–C connection or use a transit architecture. A full mesh needs **n(n − 1)/2** connections.
> - **Accepted connection + routes on both sides + SG/NACL permissions** are separate requirements. Route destination = peer CIDR; target = `pcx-...`.
> - **Same-Region** peers support security-group references, including across accounts. **Cross-Region** peers use IP/CIDR rules.
> - Peering DNS options resolve **EC2 public IPv4 hostnames to private addresses**; enable both VPC DNS attributes. They do not share a [private hosted zone](../services/route53.md#private-hosted-zone); associate the zone with the client VPC separately.
> - No transit through a peer's IGW/NAT, VPN, Direct Connect, or [gateway endpoint](../services/vpc-gateway-endpoints.md).
> - Cross-Region peering traffic stays on AWS's backbone and is encrypted before leaving AWS facilities.

### VPC Flow Logs

See [VPC Flow Logs](../services/vpc-flow-logs.md) for diagrams, fields, and troubleshooting.

> [!tip] Exam Tip
>
> - **Metadata, not payloads; delayed, not real-time.** Payload inspection requires packet capture or Traffic Mirroring.
> - Scope: **VPC → subnet → ENI**. Filters: **ACCEPT / REJECT / ALL**.
> - Destinations: **[CloudWatch Logs](../services/cloudwatch.md)**, **[S3](../services/s3.md)**, or **[Amazon Data Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose)**. SQL queries over logs stored in S3 → **Athena**.
> - An accepted ICMP request and its matching rejected reply in the illustrated scenario → check the outbound **[NACL](../services/vpc.md#network-acl-nacl)**. **[Security groups](../services/vpc.md#security-groups-sg)** automatically permit response traffic for tracked requests.
> - Do not infer “SG = one record, NACL = two”: request and response can each be logged regardless of statefulness.
> - Missing [EC2 metadata](../services/ec2.md) / DHCP / Amazon DNS traffic can reflect exclusions. Queries to your own DNS server can be logged.
> - Protocols: ICMP **1**, TCP **6**, UDP **17**. **SKIPDATA** indicates missing records, not a network rejection.

### VPC Gateway Endpoints

See [VPC Gateway Endpoints](../services/vpc-gateway-endpoints.md) for diagrams and the [S3](../services/s3.md) access-control example.

> [!tip] Exam Tip
>
> - **Private access to same-Region S3 or [DynamoDB](../services/dynamodb.md) without NAT/IGW/public client IPs** → gateway endpoint; no additional gateway-endpoint charge.
> - **Destination = AWS-managed service prefix list; target = endpoint ID.** Associate route tables, not endpoint ENIs in each subnet. Subnets using those tables inherit the route.
> - **Highly available across AZs by default**, but scoped to one [VPC](../services/vpc.md) and Region; not an endpoint that on-premises/peered/transit-connected clients can reuse.
> - **Endpoint policy** limits what can be accessed through that endpoint. **Bucket policy + explicit Deny on `aws:SourceVpce`** enforces the approved path for the selected actions; a conditional Allow alone is not an absolute restriction.
> - **Gateway endpoints do not use [PrivateLink](../services/vpc.md#aws-privatelink).** Interface endpoints use subnet [ENIs](../concepts/networking.md#elastic-network-interfaces-enis) and private IPs; both S3 and [DynamoDB](../services/dynamodb.md) also support interface endpoints.
> - Adding a gateway endpoint does not remove a default NAT/IGW route or grant missing [IAM](../services/iam.md) permissions.

### VPC Interface Endpoints

See [VPC Interface Endpoints](../services/vpc-interface-endpoints.md) and its [gateway/interface comparison](../services/vpc-interface-endpoints.md#gateway-endpoint-vs-interface-endpoint).

> [!tip] Exam Tip
>
> - **Private [SNS](../services/sns.md)/[SQS](../services/sqs.md) calls without NAT/IGW** → interface endpoint using PrivateLink, with ENIs in selected subnets and endpoint security groups.
> - **Resilience across AZs** → one endpoint resource with ENIs in multiple AZs; select one subnet per AZ. A single ENI creates an AZ dependency.
> - **Keep the application's normal service hostname** → enable supported Private DNS. AWS manages a hidden [Route 53 private hosted zone](../services/route53.md#private-hosted-zone); set **both `enableDnsSupport` and `enableDnsHostnames` to `true`**.
> - **Customer-managed SNS DNS** → private zone `sns.us-east-1.amazonaws.com`, apex A alias to the endpoint's regional DNS entry, and automatic endpoint Private DNS disabled to avoid a conflicting zone.
> - **Default name still resolves publicly** → check Private DNS/custom zone, VPC association, and the resolver used. **Private address but timeout** → check normal routes, endpoint SG inbound TCP 443, client egress, and NACLs.
> - Interface endpoints use normal routing to ENIs; they do not add a gateway-style service-prefix-list route. They charge per AZ-hour and data processed.
> - [S3](../services/s3.md) and [DynamoDB](../services/dynamodb.md) support both types, but service DNS rules differ: [DynamoDB](../services/dynamodb.md) interface clients must use endpoint-specific URLs, not the SNS private-zone recipe.

### IGW vs NAT GW

|              | IGW                           | NAT GW                         |
| ------------ | ----------------------------- | ------------------------------ |
| Direction    | Bidirectional (public subnet) | Outbound only (private subnet) |
| IP mapping   | 1:1 (private ↔ Elastic IP)    | Many-to-one                    |
| Resilience   | Regional                      | Zonal or Regional (see below)  |
| IPv6 support | Yes                           | No                             |

### Zonal vs Regional NAT GW (availability mode — as of Nov 2025)

|                         | Zonal NAT GW                          | Regional NAT GW (RNAT)              |
| ----------------------- | ------------------------------------- | ----------------------------------- |
| Scope                   | Single AZ                             | VPC-level                           |
| Public subnet required? | Yes                                   | No                                  |
| High availability       | Manual — one per AZ, per-AZ routes    | Automatic — single ID, auto-expands |
| Private NAT support     | Yes                                   | No                                  |
| Use when                | Private NAT; strict per-AZ IP control | Most public egress use cases        |

> [!tip] Exam Tip
> Pre-Nov 2025 courses only cover **zonal** NAT GW. For region resilience with zonal: deploy one per AZ, configure each AZ's route table to its local NAT GW. With **regional** NAT GW, a single ID covers all AZs automatically — no public subnet needed.

### NAT GW vs NAT Instance

- **NAT GW**: AWS-managed, no SG support, charged by duration + data
- **NAT Instance**: self-managed, supports SG, cheaper at high data volumes — must disable **source/destination checks**

### IPv4 vs IPv6

- Public IPv4 is **not configured in the OS** — only visible at the IGW level
- IPv6 is configured directly in the OS (all IPv6 is publicly routable in AWS)
- IPv6 outbound-only: use **Egress-Only IGW**, not NAT GW (NAT GW doesn't support IPv6)

### VPC Design

- Default VPC CIDR: `172.31.0.0/16` — avoid in production
- 5 reserved IPs per subnet: network, router (+1), DNS (+2), reserved (+3), broadcast (last)
- `enableDnsHostnames` = false by default on custom VPCs; set to true to get public DNS names

## EC2

See [EC2](../services/ec2.md) for full details.

- **EC2 = IaaS.** You manage OS + above; AWS manages hypervisor + below. Patching the guest OS is **your** responsibility — managed compute (Lambda, Fargate) is what shifts that line
- **EC2 is the default compute service** in AWS — pattern-match "general-purpose VM / I want an OS / long-running server" → EC2
- **Host vs Instance.** Host = physical AWS-managed hardware in one AZ. Instance = VM on a host, AZ-bound through it. Host, subnet, [ENI](../concepts/networking.md), [EBS](../services/ec2.md#storage-options) volume, and instance store are **all AZ-scoped** — no cross-AZ attachment
- **Tenancy:** **Shared** (default, hypervisor-isolated) / **Dedicated Instance** (no hardware co-tenancy, host opaque) / **Dedicated Host** (per-host allocation + per-host billing + host targeting + BYOL per-socket licensing)
- **Private by default.** An instance launches into one [VPC](../services/vpc.md) subnet; public reachability needs public subnet + [IGW](../services/vpc.md#internet-gateway-igw) + public IP + [SG](../services/vpc.md#security-groups-sg) rule. The **default VPC** wires this up automatically; **custom VPCs** require it explicitly
- **AZ-resilient, not AZ-fault-tolerant.** Instance lives in a subnet → subnet lives in an AZ → AZ failure kills the instance. HA = **fleet across multiple AZs + load balancer**, not "more durable instance"
- **Cross-AZ relocation is not native** — snapshot the EBS volume(s) or capture an AMI, then launch in the target AZ
- **State machine: Running ↔ Stopped → Terminated.** Stop ↔ Start is bidirectional; **Terminate is one-way** and irreversible
- **Stop/Start ≠ Restart.** Restart stays on the **same host** (instance store + auto public IP preserved). Stop/Start can **land on a new host** in the same AZ — instance-store data is **lost** and the auto-public-IPv4 is **released**
- **Billing granularity:** per second (Linux + most Windows AMIs, 60-second minimum) or per hour (some commercial-license AMIs)
- **AMI is region-specific** — to use it in another region, you must **copy** it (and pay the snapshot data-transfer cost). Stored in the region but **deployable to any AZ within that region** because the backing EBS snapshots live on regional S3. See [AMI — Amazon Machine Image](#ami--amazon-machine-image) for the full treatment
- **AMI permission modes:** **Private** (default, owner-only) / **Public** (everyone — e.g. AWS base images) / **Explicit** (specific account IDs granted launch permission)
- **AMI is a logical container, not a data carrier** — it holds an AMI ID + permissions + a block device mapping pointing at EBS snapshots. The real volume data lives in the snapshots; deleting the snapshots breaks the AMI
- **AMIs cannot be edited** — to change one, launch from it, modify, and create a brand-new AMI
- **Linux access: SSH/22 + private key** (direct public-key auth, no password). **Windows access: RDP/3389 + private key to decrypt the Administrator password**, then RDP with that password — the SSH key never touches the RDP session itself
- **Private key downloads once.** Lose it → create a new key pair (no recovery from AWS)

### EC2 state-based costs

| Component                   |  Running  |            Stopped             |             Terminated              |
| --------------------------- | :-------: | :----------------------------: | :---------------------------------: |
| CPU / Memory / Network      | 💰 Charged |             ✅ Free             |               ✅ Free                |
| **EBS volumes**             | 💰 Charged |      💰 **Still charged**       |          ✅ Free (deleted)           |
| Public IPv4 (auto-assigned) |   Held    |            Released            |              Released               |
| Instance-store volumes      |  Present  |            **Lost**            |                Lost                 |
| Elastic IP (attached)       |   Free    | Small hourly charge while idle | Charge continues until **released** |

### EC2 cheat sheet

| Keyword in the question                                     | Answer                                                                                                                       |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| "Stopped the instance to save money, still being billed"    | **[EBS](../services/ec2.md#storage-options) storage** still charged — only **Terminate** ends all costs                      |
| "Restore terminated instance"                               | ❌ Not possible — Terminate is irreversible; snapshot/AMI **before** terminating                                              |
| "Single instance survives AZ failure"                       | ❌ Impossible — instance is AZ-scoped; need multi-AZ fleet                                                                    |
| "Move instance to another AZ"                               | Snapshot EBS / capture AMI → launch new instance in target AZ; instances don't natively cross AZs                            |
| "Attach EBS volume from another AZ"                         | ❌ Impossible — EBS is AZ-scoped; snapshot → create volume in target AZ                                                       |
| "Multiple ENIs on different subnets"                        | OK only if the subnets are in the **same AZ**                                                                                |
| "Rebooted, lost scratch disk"                               | Was it really a reboot? **Stop/Start** can land on a new host → **instance-store lost**. Restart keeps the same host         |
| "Public IP changed after I cycled the instance"             | Stop/Start releases auto public IPv4; use an **Elastic IP** for stability                                                    |
| "Use the same AMI in another region"                        | **Copy AMI** to the destination region first (`aws ec2 copy-image`); copied AMIs default back to **Private** permissions     |
| "Use the same AMI in another AZ"                            | **No copy needed** — an AMI deploys to any AZ in its home region                                                             |
| "Edit an existing AMI to add a package"                     | **Cannot edit** — launch from it, configure, capture a new AMI                                                               |
| "Cost of an AMI sitting in an account"                      | Billed on the **EBS snapshots' used data**, not the AMI container or the volume's allocated size                             |
| "Marketplace AMI billing"                                   | Normal instance hourly rate **plus** a per-hour software-license charge bundled into the AMI                                 |
| "Build many identical web servers from a configured master" | **AMI baking** — configure one, capture as AMI, launch the fleet                                                             |
| "Per-socket Windows / Oracle / SQL Server licensing"        | **Dedicated Host** — pay per host, target the host, satisfy BYOL per-socket licensing                                        |
| "No hardware co-tenancy but I don't care about the host"    | **Dedicated Instance** — single-tenant hardware, host stays opaque                                                           |
| "Instance has no internet despite being in public subnet"   | Check public IP assignment, SG inbound, route to [IGW](../services/vpc.md#internet-gateway-igw), NACL ephemeral return       |
| "Connect to Windows EC2"                                    | RDP/3389 + use **private key to decrypt admin password**                                                                     |
| "Connect to Linux EC2"                                      | SSH/22 + private key (direct auth, no password)                                                                              |
| "Lost the SSH key"                                          | Create a new key pair; detach root volume → attach to recovery instance to re-key (no AWS recovery)                          |
| "Launch many instances from a configured image"             | Create an **AMI** from the configured instance, then launch from it                                                          |
| "Share an AMI with another AWS account"                     | Set permission to **Explicit** with that account's ID                                                                        |
| "Instance-level firewall"                                   | [Security Group](../services/vpc.md#security-groups-sg) on the [ENI](../concepts/networking.md) — not a subnet-level control |
| "Service role for EC2 to call S3 / KMS / etc."              | [IAM role](../services/iam.md#iam-roles) attached to the instance — **never embed access keys**                              |
| "Per-second vs per-hour billing"                            | Per-second (60s minimum) on most Linux + Windows; per-hour on certain commercial AMIs                                        |
| "Long-running 24/7 server with a specific OS"               | **EC2** (Lambda has a 15-min runtime cap; containers / Beanstalk are deviations that require a reason)                       |

> [!tip] Exam Tip
> Pattern-match: *"reduce cost of an idle instance overnight"* → **stop** it (saves compute) — but note **storage still bills**. *"reduce cost permanently"* → **terminate** + snapshot first if data matters. *"instance fails, AZ failed"* → expected behaviour; HA is a **multi-AZ fleet** concern, not an instance property. *"AMI not available in region X"* → **copy** AMI from source region. *"cannot SSH but security group + route are correct"* → check **NACL ephemeral return ports** (NACLs are stateless). *"scratch disk gone after cycling the instance"* → it was a **stop/start**, not a restart; the instance may have moved to a new host. *"BYOL with per-socket licensing"* → **Dedicated Host**.

> [!warning] Common EC2 Gotchas
>
> - **Stopped ≠ free.** EBS volumes keep billing while stopped — the most-tested cost trap
> - **Terminate deletes EBS root volume by default** (`DeleteOnTermination=true`). Snapshot first if you need the data
> - **Instance-store data is ephemeral** — lost on stop, lost on terminate, lost on host failure. Don't use it for anything you can't lose
> - **Stop/Start ≠ Restart.** Stop/Start can move the instance to a new host (same AZ) → loses instance-store data and releases the auto public IPv4. Restart keeps the same host
> - **Everything is AZ-scoped** — host, subnet, ENI, EBS, instance store. You cannot attach an EBS volume or ENI from one AZ to an instance in another AZ
> - **Dedicated Instance vs Dedicated Host** look similar but are not the same: Dedicated Host adds **per-host billing**, **host targeting**, and **BYOL per-socket licensing**; Dedicated Instance only guarantees no hardware co-tenancy
> - **AMI ≠ snapshot.** An AMI references one or more EBS snapshots plus metadata (permissions, block-device mapping); deleting just the underlying snapshots breaks the AMI
> - **Region-locked AMIs.** Multi-region launch requires `aws ec2 copy-image` first; otherwise the AMI ID won't exist in the target region
> - **AMIs cannot be edited.** The only path is (1) launch from the AMI, (2) modify the instance, (3) capture a new AMI; the old AMI is immutable
> - **Marketplace AMIs add a per-hour license charge** on top of the standard EC2 instance rate — easy to miss when sizing cost for vendor-bundled stacks
> - **Private key is download-once.** A lost private key on a single-key-pair instance means a painful recovery via root-volume detach

### Containers Fundamentals

See [Containers](../concepts/containers.md) for the full concept page.

- **Container vs VM:** a VM boots a full guest OS on a hypervisor; a container runs as an isolated process environment on a host OS. The container packages app code, libraries, and user-space dependencies, but shares the host kernel
- **Container engine:** Docker is the common name, but Docker is one implementation. "Docker container" is not synonymous with every possible container
- **Image vs container:** an image is the template; a container is a running instance of that image with a runtime read/write layer
- **Image layers are read-only.** A Dockerfile builds an image layer-by-layer. Each step stores only the filesystem change it introduced
- **Runtime R/W layer:** the per-container layer holds runtime changes such as logs and temporary files. Containers from the same image share the read-only layers and differ mostly by this layer
- **Registry:** images are pushed to and pulled from registries. Docker Hub is a public registry; AWS container service designs use the same registry/image/host/run vocabulary
- **Benefits:** portability, consistency, density, fast start/stop, and less duplicated OS overhead than VMs
- **Ports:** exposing a container port is only the container-level side. Real reachability still depends on host/task networking, load balancers, [VPC](../services/vpc.md) routes, and [Security Groups](../services/vpc.md#security-groups-sg)

#### Containers cheat sheet

| Keyword in the question                         | Answer                                               |
| ----------------------------------------------- | ---------------------------------------------------- |
| "Runs as a process, not a full OS"              | **Container**                                        |
| "Full guest OS, kernel, and boot lifecycle"     | **VM / EC2 instance**                                |
| "Package app plus dependencies for portability" | **Container image**                                  |
| "Build instructions for the image"              | **Dockerfile**                                       |
| "Read-only filesystem layers"                   | **Image layers**                                     |
| "Per-container runtime filesystem changes"      | **Read/write layer**                                 |
| "Where images are pushed and pulled"            | **Container registry**                               |
| "Docker Hub"                                    | A public registry, not the whole container ecosystem |
| "Same image, many running copies"               | Many containers sharing the image layers             |

> [!tip] Exam Tip
> Pattern-match: portability, consistency, high density, and fast startup point to containers. Full OS control, per-socket licensing, or traditional VM migration points back to [EC2](../services/ec2.md#when-to-use-ec2).

#### Container gotchas

> [!warning] Common Container Gotchas
> A base image is not a full OS kernel; it contains user-space files and libraries, while the running container still uses the host OS kernel. Image layers are read-only, runtime changes belong in the per-container read/write layer, and that layer is not durable application state. Exposed ports also do not equal reachable services; AWS networking controls still decide whether traffic can arrive.

### ECS

See [ECS](../services/ecs.md) for the full service page and [Containers](../concepts/containers.md) for container fundamentals.

- **ECS is the managed orchestrator** for containers. It runs tasks from task definitions and keeps production workloads healthy through services
- **Hierarchy:** cluster contains services; services manage tasks; tasks contain one or more containers
- **Container definition:** image, ports, and container-level behavior
- **Task definition:** application template containing one or more container definitions plus CPU, memory, networking, compatibility, and roles
- **Task role:** [IAM role](../services/iam.md#iam-roles) assumed by the task. This is the best-practice way for application containers to access AWS services
- **Task execution role:** platform/setup role used by ECS or Fargate to pull images and write logs. Do not confuse it with the task role
- **Service:** long-running distributed layer that keeps a desired number of task copies running, replaces failed tasks, supports rolling deployments, and can attach to a load balancer
- **ECR:** AWS-managed container registry (see the dedicated [ECR section](#ecr) below); ECS task definitions can point at images in [ECR](../services/ecr.md) or another registry
- **EC2 launch type:** ECS places tasks on EC2 container instances in your account. You manage and pay for the host fleet even when it is idle, but you get EC2 pricing flexibility such as Spot and Reserved Instances
- **Fargate launch type:** no EC2 host fleet to manage. Tasks run on AWS-managed infrastructure and attach to your [VPC](../services/vpc.md) through [ENIs](../concepts/networking.md#elastic-network-interfaces-enis)
- **Fargate billing correction:** billed for configured task resources while tasks run, not for live CPU utilization only

#### ECS cheat sheet

| Keyword in the question                                                          | Answer                               |
| -------------------------------------------------------------------------------- | ------------------------------------ |
| "Containerized app on AWS, managed orchestration"                                | **ECS**                              |
| "Where does ECS get the image?"                                                  | **Container registry**, commonly ECR |
| "One running copy of an application"                                             | **Task**                             |
| "Application template with one or more containers"                               | **Task definition**                  |
| "One container inside a task definition"                                         | **Container definition**             |
| "Keep 5 copies running and replace failed copies"                                | **ECS service**                      |
| "Container app needs S3 / [DynamoDB](../services/dynamodb.md) / KMS permissions" | **ECS task role**                    |
| "Pull image from ECR and write logs"                                             | **Task execution role**              |
| "Large, steady, price-sensitive container workload"                              | **ECS on EC2**                       |
| "Small / bursty / periodic / no host management"                                 | **ECS on Fargate**                   |
| "Serverless but still has VPC networking"                                        | **Fargate task with ENI**            |

> [!tip] Exam Tip
> Pattern-match ECS launch type questions by the management line. If the company wants host control, Spot / Reserved economics, or dense packing for steady load, use ECS on EC2. If the company wants no host management or has small, bursty, or periodic container work, use ECS on Fargate.

> [!warning] Common ECS Gotchas
> A standalone task is not highly available by itself; use a service for production desired-count management. Fargate removes EC2 host management, but it does not remove VPC design: subnets, security groups, routes, public IP assignment, and load balancers still matter. The task role is for app permissions; the execution role is for ECS platform setup.

### ECR

See [ECR](../services/ecr.md) for the full service page.

- **ECR is the managed container image registry** — "Docker Hub for AWS." It stores, secures, scans, and distributes images; it does not build or run them
- **Hierarchy:** registry → repository → image → tag. **Tags are unique within a repository**
- **One private registry per account, per region** (endpoint `<account>.dkr.ecr.<region>.amazonaws.com`); plus access to a public registry. The private registry is a **regional** resource
- **Public registry:** anyone can pull, push needs permissions. **Private registry:** permissions required for **all** operations
- **[IAM](../services/iam.md) controls all access**; **repository policies** (resource policies) enable cross-account pulls
- **Enhanced scanning = Amazon Inspector**, OS + software packages, layer-by-layer; **Basic** is the lighter built-in scan
- **Observability:** CloudWatch (metrics), CloudTrail (API audit), EventBridge (image-change events)
- **Replication is cross-region AND cross-account**
- **Who pulls the private image?** On ECS it's the **[task execution role](../services/ecs.md#task-definitions-and-task-roles)**, not the task role

#### ECR cheat sheet

| Keyword in the question                                            | Answer                                           |
| ------------------------------------------------------------------ | ------------------------------------------------ |
| "Managed container image registry on AWS"                          | **ECR**                                          |
| "Tags must be unique within…"                                      | The **repository**                               |
| "Anyone can pull, no credentials"                                  | **Public** registry                              |
| "Permissions required for pull and push"                           | **Private** registry                             |
| "Grant another AWS account pull access"                            | **Repository (resource) policy**                 |
| "CVE / vulnerability scan, OS + language packages, layer-by-layer" | **Enhanced scanning (Inspector)**                |
| "Lighter, default image scan"                                      | **Basic scanning**                               |
| "Metrics on push / pull / auth"                                    | **CloudWatch**                                   |
| "Audit who called which ECR API"                                   | **CloudTrail**                                   |
| "Trigger a workflow when a new image is pushed"                    | **EventBridge**                                  |
| "Copy images to another region or account"                         | **Replication** (cross-region and cross-account) |
| "ECS can't pull the private image"                                 | Check the **task execution role** permissions    |

> [!tip] Exam Tip
> The two highest-value ECR facts: **Enhanced scanning is the Inspector tier** (OS + software packages, layer-by-layer) — Basic is the lighter option; and **ECR replication is both cross-region and cross-account**. For "who pulls the image on ECS," the answer is the **task execution role**.

> [!warning] Common ECR Gotchas
> The private registry is **regional** — an image pushed to one region isn't in another unless you configure replication. Public vs private only changes the **read** default (anonymous pull on public); push always needs permissions. Don't credit **Inspector** to Basic scanning — only **Enhanced** uses it.

### Kubernetes & EKS

See [Kubernetes](../concepts/kubernetes.md) for the cloud-agnostic concept and [EKS](../services/eks.md) for the AWS service.

- **Kubernetes** is an open-source, **cloud-agnostic** container orchestrator. Control plane does the thinking; **nodes** run the workloads
- **Control-plane components:** **kube-apiserver** (front door, everything talks to it), **etcd** (key/value state store), **kube-scheduler** (assigns pods to nodes), **kube-controller-manager** (controllers, e.g. Node Controller responds to node outages), **cloud-controller-manager** (optional cloud integration)
- **Node software:** container runtime, **kubelet** (talks to control plane via the K8s API), and **kube-proxy** (runs on every node, implements services) — kube-proxy is a **node** component, not control-plane
- **Pod** = smallest unit; **think pods, not containers**; one-container-one-pod is the common pattern. **Pods are non-permanent and not highly available** — services provide the stable abstraction
- **Service** = consistent abstraction over pods (the "application"). **Job** = run-to-completion task. **Ingress** = external access to a service; the **Ingress Controller** provisions the hardware (on AWS, the AWS Load Balancer Controller uses ALB/NLB)
- **Storage:** default is **ephemeral and node-local** (like [EC2 instance store](../services/ec2.md#instance-store), lost on pod relocation); **Persistent Volumes** outlive pods
- **EKS = AWS-managed Kubernetes** — same Kubernetes, AWS runs it. Pick it for **cloud-agnostic / avoid lock-in / existing Kubernetes** workloads
- **Control plane AND etcd** are both AWS-managed and **multi-AZ**
- **Node options:** self-managed EC2, **managed node groups** (EC2), and **Fargate** (serverless, via **Fargate profiles**)
- **Integrations:** ECR (images), ELB (load balancing/ingress), IAM (security), VPC (networking)
- **Persistent storage providers:** EBS, EFS, FSx
- **Architecture:** control plane lives in an **AWS-managed VPC** (multi-AZ); EC2 workers live in **your VPC**; connectivity via **injected ENIs** or the **public endpoint**

#### Kubernetes & EKS cheat sheet

| Keyword in the question                                 | Answer                                           |
| ------------------------------------------------------- | ------------------------------------------------ |
| "Container orchestration," open standard, avoid lock-in | **Kubernetes / EKS**                             |
| "Front end for the control plane"                       | **kube-apiserver**                               |
| "Cluster key/value backing store"                       | **etcd**                                         |
| "Assigns pods to nodes"                                 | **kube-scheduler**                               |
| "Responds to node outages"                              | **Node Controller** (in kube-controller-manager) |
| "Network proxy on every node"                           | **kube-proxy** (node component)                  |
| "Consistent abstraction over pods / the app"            | **Service**                                      |
| "External access to a service"                          | **Ingress** (+ Ingress Controller)               |
| "Run to completion, then finish"                        | **Job**                                          |
| "Storage that outlives a pod"                           | **Persistent Volume** (EBS/EFS/FSx on EKS)       |
| "Managed Kubernetes on AWS"                             | **EKS**                                          |
| "Already run Kubernetes elsewhere, want managed AWS"    | **EKS**, not ECS                                 |
| "Serverless EKS nodes"                                  | **Fargate** via a **Fargate profile**            |
| "EKS worker nodes — where do they run?"                 | **Your (customer) VPC**                          |
| "EKS control plane — where does it run?"                | **AWS-managed VPC**, multi-AZ                    |

> [!tip] Exam Tip — ECS vs EKS
> Both run on EC2 or Fargate, so launch type is **not** the differentiator. Choose **EKS** when the scenario wants Kubernetes specifically (cloud-agnostic, open standard, existing K8s workload). Choose **ECS** when an AWS-native orchestrator is fine and you want to avoid EKS's **per-cluster control-plane hourly fee** (ECS has none).

> [!warning] Common Kubernetes & EKS Gotchas
> Pods are **not** durable or highly available — never design around a specific pod surviving. **kube-proxy and kubelet are node components**, not control-plane. Default pod storage is **ephemeral/node-local** (instance-store-like), not EBS-durable. On EKS, **both the control plane and etcd** are multi-AZ — don't credit only the control plane. EKS worker nodes sit in **your VPC**, while the control plane sits in an **AWS-managed VPC**. EKS bills an **hourly per-cluster** control-plane fee that ECS does not — a real cost reason to prefer ECS when Kubernetes isn't required.

### EC2 Status Checks & Auto Recovery

See [EC2 — Status Checks and Auto Recovery](../services/ec2.md#status-checks-and-auto-recovery) for full details.

- **Every EC2 instance has two independent status checks**: **System** (the EC2 service and host underneath) and **Instance** (the OS layer). A healthy, fully booted instance shows **2/2 checks passed** — anything else after boot completes is a real problem
- **System check fails** → AWS-side issue (host power, host network, host hardware/software). Common manual fixes that work: **Stop/Start** (lands on a new host) or **Recover** (auto recovery)
- **Instance check fails** → OS-side issue (corrupt filesystem, OS networking misconfig, kernel/boot problem). Reboot, console-based recovery, or rebuild from AMI
- **The static-public-IPv4 trap** — setting a public IPv4 on an OS interface is a classic instance-check failure pattern. The public IP lives on the [IGW](../services/vpc.md#internet-gateway-igw) as 1:1 NAT, never inside the guest
- **Status Check Alarm** = a [CloudWatch](../services/cloudwatch.md) alarm created via the EC2 console (*Actions → Create Status Check Alarm*). Fires when **either** check fails for **≥ 1 consecutive 5-minute period**
- **Alarm destinations**: SNS topic (notification) **plus** one optional EC2 action — **Reboot**, **Stop**, **Terminate**, or **Recover** (auto recovery)
- **Terminate as an alarm action** only "self-heals" if paired with an [Auto Scaling group](../patterns/scaling.md) that re-provisions a replacement — on a lone instance, Terminate is just deletion
- **Auto recovery (the Recover action)** brings the instance back to life by anything from a restart to migrating it to a **brand-new EC2 host**. **All IP addressing is preserved** — primary private IPv4, the auto-assigned public IPv4, and any Elastic IP. With software set to auto-start, the heal is fully unattended
- **Auto recovery requirements**: (1) **same AZ only** — does not protect against AZ failure, only isolated host/instance failures; (2) **supported instance type** (older docs cite A1/C4/C5/M4/M5/R3/R4/R5 — the modern list is much broader); (3) **EBS-only** — instance store would be lost on host migration; (4) **spare host capacity** must exist in the AZ
- **Simplified Automatic Recovery** is the modern default — most current-generation types are auto-recovered without you setting up a CloudWatch alarm. The mechanism and limits are identical; the difference is opt-out vs opt-in

#### Auto Recovery vs Stop/Start vs Restart cheat sheet

| Operation            | Host migration               |       Instance-store data       | Auto public IPv4               | Elastic IP |
| -------------------- | ---------------------------- | :-----------------------------: | ------------------------------ | :--------: |
| **Restart (reboot)** | ❌ Never — same host          |            Preserved            | Preserved                      | Preserved  |
| **Stop → Start**     | Possible (new host, same AZ) |            **Lost**             | **Released — new IP on Start** | Preserved  |
| **Auto recovery**    | Possible (new host, same AZ) | **Lost** (EBS-only requirement) | **Preserved**                  | Preserved  |

#### Status Checks & Auto Recovery cheat sheet

| Keyword in the question                                                           | Answer                                                                                                                                                                      |
| --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Healthy instance shows X/Y checks"                                               | **2/2 checks passed**                                                                                                                                                       |
| "The check that failed indicates a host hardware fault"                           | **System status check**                                                                                                                                                     |
| "The check that failed indicates an OS-level issue / corrupt FS / kernel problem" | **Instance status check**                                                                                                                                                   |
| "Static public IPv4 set on the OS interface"                                      | ❌ Instance check failure pattern — public IPv4 is NAT'd at the IGW, never in the guest                                                                                      |
| "How do I get notified when a check fails"                                        | **CloudWatch Status Check Alarm → SNS topic**                                                                                                                               |
| "Alarm triggers when…"                                                            | Either check fails for **at least one consecutive 5-minute period**                                                                                                         |
| "Optional EC2 actions on the alarm"                                               | **Reboot / Stop / Terminate / Recover**                                                                                                                                     |
| "Self-heal a failed instance, keep its IP, no manual action"                      | **Recover** (auto recovery)                                                                                                                                                 |
| "Auto recovery preserves which IPs?"                                              | **All of them** — primary private IPv4, auto-assigned public IPv4, **and** Elastic IP                                                                                       |
| "Auto recovery and AZ failure"                                                    | ❌ Same-AZ only — does not survive AZ-wide failure. Use **multi-AZ + ELB + Auto Scaling** for HA                                                                             |
| "Auto recovery on an instance with instance-store volumes"                        | ❌ Not supported — **EBS-only** requirement                                                                                                                                  |
| "Auto recovery fails during a major regional event"                               | Likely **no spare host capacity** in the AZ — recovery depends on AWS having a target host to migrate to                                                                    |
| "Auto recovery isn't working on this old instance type"                           | Verify the type is supported — older list **A1, C4, C5, M4, M5, R3, R4, R5**; modern types are broader and most are recovered automatically (Simplified Automatic Recovery) |
| "Terminate action on a single instance — does it replace?"                        | ❌ No — Terminate **deletes**; replacement requires pairing with [Auto Scaling](../patterns/scaling.md)                                                                      |

> [!tip] Exam Tip
> Pattern-match: *"check the host underneath"* → **System status check**. *"check the OS inside"* → **Instance status check**. *"automated heal that preserves the same IP"* → **Recover** (auto recovery), not Stop/Start. *"automated heal that survives AZ failure"* → ❌ not auto recovery — you need a **multi-AZ fleet behind a load balancer**. *"alarm fires after 5 minutes of failed checks"* → standard Status Check Alarm trigger. *"notify the team when an instance becomes unhealthy"* → **CloudWatch Status Check Alarm + SNS**.

> [!warning] Common Status Check / Auto Recovery Gotchas
>
> - **Auto recovery is single-AZ, not HA.** It saves you from an isolated host failure, not from an AZ outage. The exam trap is selecting "enable auto recovery" as the answer to "design for AZ failure"
> - **Auto recovery requires EBS-only.** Instance-store volumes block recovery — by definition, local disks cannot follow the instance to a new host
> - **Auto recovery requires spare capacity.** In a major regional event, the underlying mechanism can silently fail because AWS has no free host to migrate to
> - **Terminate as an alarm action is not a replacement mechanism on its own** — only when paired with Auto Scaling does termination self-heal. On a lone instance, Terminate ends the instance and that's it
> - **5 minutes of consecutive failure** is the trigger — shorter blips don't fire the alarm. Use richer CloudWatch metric thresholds if you need a tighter SLO
> - **Auto recovery preserves IPs, including the auto-assigned public IPv4** — this is the one case where a host-migration event does *not* release the dynamic public IP. The contrast with Stop/Start is exam-testable

### EC2 Instance Types

See [EC2 — Instance Types](../services/ec2.md#instance-types) for full details.

- The whole string (e.g. `R5dn.8xlarge`) is the **instance type** — use it in full; sub-components: family letter + generation + optional capability letters + size
- **Category = workload bottleneck**: C = CPU, R = RAM, I/D = local storage throughput, G/P = GPU/parallel processing, T/M = general purpose (burstable vs steady-state)
- **Additional capability letters** between the generation and size: `a` = AMD, `d` = NVMe local disk, `n` = network optimised, `e` = extra RAM/storage, `g` = Graviton (ARM)
- **Prefer the newest generation** within a family — better price-to-performance. Only deviate if the region lacks it or approvals require otherwise
- EBS-heavy workloads: the **instance's storage network bandwidth cap** is an independent ceiling from the EBS volume's provisioned IOPS — check both

#### Instance category quick-pick

| Category                  | Keyword in the question                                                               | Typical families               |
| ------------------------- | ------------------------------------------------------------------------------------- | ------------------------------ |
| **General Purpose**       | "default", "balanced", "burst", "low average load"                                    | T3, T3a, M5, M5a, M5n, A1, M6g |
| **Compute Optimized**     | "CPU-intensive", "HPC", "media encoding", "gaming server", "scientific modelling"     | C5, C5n                        |
| **Memory Optimized**      | "in-memory database", "cache", "Redis", "real-time analytics", "large dataset in RAM" | R5, R5a, X1, z1d, High Memory  |
| **Accelerated Computing** | "GPU", "ML training/inference", "FPGA", "parallel processing", "genomics"             | P3, G4, F1, Inf1               |
| **Storage Optimized**     | "NoSQL", "Elasticsearch", "data warehouse", "high IOPS local storage", "Hadoop/HDFS"  | I3/I3en, D2, H1                |

> [!tip] Exam Tip
> Pattern-match: *"T3 performance degraded over time under consistent load"* → **CPU credit exhaustion** — switch to **M5** for sustained CPU workloads. *"max RAM per dollar / in-memory workload"* → **R5 / X1**. *"GPU / ML training"* → **P3 / G4**. *"programmable hardware / genomics / financial analysis"* → **F1 (FPGA)**. *"large NVMe SSD, NoSQL or analytics"* → **I3/I3en**. *"Hadoop / HDD-heavy throughput"* → **D2 / H1**. *"ARM-native, cost-optimised"* → **A1 / M6g (Graviton)**.

> [!warning] Gotcha — T3 burstable vs M5 steady-state
> **T3** instances accrue CPU credits at idle and spend them under load. Once credits run out, CPU is throttled to baseline. The trap: a T3 performs well initially then slows down. This is **not** an infrastructure failure — it is the burstable billing model working as designed. **M5** (non-burstable) is the correct fix for sustained steady-state CPU workloads.

### Instance Store

See [EC2 — Instance Store](../services/ec2.md#instance-store) and [EBS vs Instance Store](../services/ebs.md#ebs-vs-instance-store--decision-framework) for full details.

- Instance store volumes are **physically attached to the EC2 host** — no network hop → **highest storage performance in AWS**
- **Ephemeral** — data is lost when the instance moves to a new host: stop/start, resize, host failure, or volume hardware failure
- **Reboot preserves instance store data** — a restart stays on the same host. The trap: *"rebooted and lost scratch data"* is almost always a **stop/start in disguise**
- **Launch time only** — instance store cannot be added after the instance is running
- **Included in instance price** — no cost reason to avoid using already-allocated volumes
- **Best-in-class performance**: D3 → ~4.6 GB/s HDD; i3 → ~16 GB/s NVMe SSD, ~2 million read IOPS

#### Instance store vs EBS — decision table

| If the question emphasizes…               | Pick                                       |
| ----------------------------------------- | ------------------------------------------ |
| Persistence / durability                  | **EBS**                                    |
| Resilience within an AZ + snapshots       | **EBS**                                    |
| Detach & reattach to another instance     | **EBS**                                    |
| Resilience **with** app-level replication | **Instance store** (across many instances) |
| Lowest cost HDD                           | **ST1 / SC1**                              |
| Throughput / streaming                    | **ST1**                                    |
| Boot volume                               | **Not ST1/SC1**                            |
| 16,000–64,000 IOPS per volume             | **IO1 / IO2**                              |
| Up to 256,000 IOPS per volume             | **IO2 Block Express**                      |
| Max per instance (RAID 0 over EBS)        | **~260,000 IOPS**                          |
| More than 260,000 IOPS, non-persistent OK | **Instance store** (i3: ~2 million IOPS)   |

> [!tip] Exam Tip
> Pattern-match: *"highest possible storage performance, data loss acceptable"* → **instance store**. *"app replicates its own data (Cassandra, HDFS)"* → **instance store** across many instances — let the app provide durability, keep the speed and the cost saving. *"need more than ~260,000 IOPS per instance"* → **instance store** (RAID 0 over EBS is capped at ~260,000; i3 reaches millions). *"add volumes after launch"* → only **EBS** — instance store is **launch-time only**. *"D3 for sequential throughput"* → instance store (~4.6 GB/s) beats any EBS HDD. *"i3 for NVMe IOPS"* → instance store (~2 million IOPS) vs IO2 Block Express (256,000/volume, ~260,000/instance).

> [!warning] Common Instance Store Gotchas
>
> - **Reboot preserves, stop/start does not.** "Lost scratch data after a reboot" = almost always a stop/start; the instance landed on a new host
> - **Launch time only** — cannot attach instance store after launch; any question about adding post-launch is testing this
> - **Not all instance types include instance store** — verify instance type specs before assuming
> - **Included in price** — no reason to leave allocated volumes unused; the cost is already sunk

### EC2 Network Interfaces (ENIs) & Instance DNS

See [EC2 — Network Interfaces (ENIs)](../services/ec2.md#network-interfaces-enis) for full details.

- **Network identity lives on the ENI, not the instance.** Primary private IPv4, secondary private IPv4s, optional public IPv4, public/private DNS names, MAC, EIPs, IPv6, Security Groups, and source/destination check are **all per-ENI**
- **Every instance has exactly 1 primary ENI + 0 or more secondary ENIs.** The primary ENI **cannot be detached**; secondary ENIs **can** be detached and re-attached to another instance — the MAC, IPs, EIPs, and SGs travel with the ENI
- **All ENIs on one instance must be in the same AZ.** They can sit in **different subnets**, but those subnets must all belong to the same AZ — there is no cross-AZ ENI
- **Internal DNS name format**: `ip-<private-ipv4-dashed>.ec2.internal` (e.g. `ip-10-16-0-10.ec2.internal`) — resolvable only **inside the VPC**, always to the **private** IP
- **Public DNS name format**: `ec2-<public-ipv4-dashed>.compute-1.amazonaws.com` — and resolves with **split-horizon** behaviour
- **The auto-assigned public IPv4 is NOT visible inside the OS** — 1:1 NAT applied at the [IGW](../services/vpc.md#internet-gateway-igw). Configure only the **private** IPv4 in the guest. **IPv6 is OS-visible** (and all IPv6 in AWS is publicly routable)
- **The auto public IPv4 is dynamic** — released on **Stop**, a **new one** assigned on **Start**, preserved across **Reboot**. *"My public IP changed after I rebooted"* → the action was a **Stop/Start**, not a reboot
- **Attaching an EIP removes the dynamic public IPv4**; detaching the EIP later gives a **new, different** dynamic public IP — the original is **lost for good**
- **Source/destination check** is enabled by default — **must be disabled** for [NAT instances](../services/vpc.md#nat-gw-vs-nat-instance), software routers, or any in-line forwarding role
- **Security Groups attach to ENIs.** A SG governs **every IP on the ENI**. Need different SGs for different IPs on the same instance → split them across **multiple ENIs**
- **Movable secondary ENI = MAC-licensing & IP-failover pattern.** Bind the licence (or service IP + EIP) to a secondary ENI; re-attach it to a replacement instance instead of re-issuing or re-DNS-ing

#### Split-horizon DNS — the most exam-tested ENI behaviour

The EC2 public DNS name resolves **differently depending on caller location**:

| Caller          | Public DNS name resolves to |
| --------------- | --------------------------- |
| Inside the VPC  | **Private IPv4**            |
| Outside the VPC | **Public IPv4**             |

One hostname works everywhere; **intra-VPC traffic stays private** (never hairpins through the IGW). This is what makes VPC peering and hybrid networking work cleanly when apps connect by public DNS name. See [DNS — Split-Horizon DNS](../concepts/dns.md#split-horizon-dns).

#### ENI / DNS / EIP cheat sheet

| Keyword in the question                                                    | Answer                                                                                                           |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| "Security Group attaches to…"                                              | **ENI**, not the instance or subnet                                                                              |
| "Different SGs for different IPs on one instance"                          | Split IPs across **multiple ENIs**                                                                               |
| "Multiple ENIs on different subnets"                                       | Allowed only if those subnets are in the **same AZ**                                                             |
| "Move the licence with the server when I replace it"                       | **Secondary ENI** with the MAC-bound licence + re-attach on the replacement                                      |
| "Stable public IPv4 across stop/start"                                     | **Elastic IP** (EIP)                                                                                             |
| "Detached my EIP and the original public IP came back"                     | ❌ Wrong — a **new, different** dynamic public IP is assigned; the original is lost                               |
| "Public IP changed after rebooting"                                        | Likely a **Stop/Start in disguise** — reboot preserves the auto public IPv4                                      |
| "Internal DNS name doesn't resolve from on-prem"                           | `ip-…ec2.internal` is **VPC-only**; use Route 53 Resolver inbound endpoints or split-view PHZ for hybrid         |
| "Public DNS name resolves to a private IP from inside the VPC"             | **Split-horizon DNS** — intended behaviour                                                                       |
| "Apps in two peered VPCs connect via public DNS — does traffic leave AWS?" | **No** — split-horizon DNS resolves to the **private IP** on both sides                                          |
| "EC2 acting as a NAT instance drops packets"                               | **Source/destination check** is still enabled — disable it on the NAT instance's ENI                             |
| "Configure the public IPv4 inside the guest OS"                            | ❌ Not possible — public IPv4 is invisible to the OS; only IPv6 is OS-visible                                     |
| "Cannot SSH after stop/start, OS firewall unchanged"                       | New auto public IPv4 was assigned — update your SSH target (or attach an **EIP** for stability)                  |
| "Need every existing public IP to remain forever"                          | **Allocate an EIP** *before* detaching anything — otherwise the dynamic IP is permanently lost on first EIP swap |

> [!tip] Exam Tip
> Pattern-match by ownership: *"X is attached to the instance"* — read it as *"X is attached to the instance's primary ENI."* The handful of exceptions matter: things that live on the **EC2 host** (instance store, host-affinity, dedicated-host BYOL) or in the **subnet/AZ** ([NACLs](../services/vpc.md#network-acl-nacl), [route tables](../services/vpc.md#route-tables)) are *not* on the ENI. Everything else — IPs, DNS names, MAC, EIPs, IPv6, SGs, source/dest check — is per-ENI.

> [!warning] Common ENI / DNS / EIP Gotchas
>
> - **EIP detach is a one-way IP swap.** Disassociate an EIP → a new dynamic public IPv4 is assigned, not the original. If you need the original, keep the EIP attached
> - **Public IPv4 invisibility in the OS** — only the private IPv4 is seen by the guest; the IGW does 1:1 NAT. IPv6 is the lone exception (OS-visible, all publicly routable)
> - **Reboot ≠ Stop/Start** for the auto public IPv4 — reboot preserves it; Stop/Start releases it and assigns a new one
> - **Source/destination check defaults on** — silently breaks NAT instances and any in-line forwarder until you turn it off
> - **MAC is per-ENI, not per-instance** — re-attaching a secondary ENI carries the MAC. Useful for MAC-locked licensing; also why the *primary* ENI's MAC is fixed for the life of the instance
> - **Internal DNS is VPC-scoped** — `ip-…ec2.internal` does not resolve from outside the VPC; use Route 53 private hosted zones + resolver endpoints for hybrid resolution

### EC2 Instance Metadata Service (IMDS)

See [EC2 — Instance Metadata Service (IMDS)](../services/ec2.md#instance-metadata-service-imds) for full details.

- **The endpoint is a memorizable URL**: `http://169.254.169.254/latest/meta-data/<attribute>` — link-local IPv4, **same address on every EC2 instance**. Memorize as "169.254 repeated" + "latest meta-data"
- **`169.254.0.0/16` is the link-local range** — the request never leaves the instance, so IMDS is reachable from inside even with no internet route or public IP
- **IMDS is the only way for code on the instance to discover its own public IPv4** — the OS configures only the private IPv4; the public IPv4 lives on the [IGW](../services/vpc.md#internet-gateway-igw) as 1:1 NAT. *"How does the app on the instance learn its public address?"* → **IMDS**, not `ifconfig` / `ipconfig`
- **IMDS is how the SDK gets IAM role credentials** — when an instance has an [IAM role](../services/iam.md#iam-roles) (instance profile) attached, the temporary creds (access key + secret + **session token** + expiry) live at `iam/security-credentials/<role-name>` and are refreshed automatically. This is the mechanism that makes *"never bake access keys into AMIs or user data"* possible
- **User data is exposed via IMDS** at `/latest/user-data/` (separate from `/meta-data/`) — bootstrap scripts can read what the launch passed them
- **EC2 Instance Connect injects short-lived SSH public keys via IMDS** — the SSH daemon's helper script pulls them out at login time
- **IMDSv1 has no authentication and no encryption** — anything with a shell on the instance, or any SSRF vulnerability in a web app on the instance, can read everything IMDS exposes including the **role credentials**. This is the Capital One 2019 breach pattern
- **IMDSv2 fixes this with session tokens** — clients first `PUT` to `/latest/api/token` (with `X-aws-ec2-metadata-token-ttl-seconds`), then send the returned token in `X-aws-ec2-metadata-token` on every subsequent `GET`. SSRF cannot obtain the token because most HTTP libraries forced into making a `GET` cannot pivot to a `PUT`
- **Enforce IMDSv2 with `HttpTokens=required`** — per-instance setting; modern hardening baseline. `optional` accepts both v1 and v2
- **The default version is set by the AMI's `imds-support` parameter, not the OS** — **Amazon Linux 2** leaves it unset, so instances default to `optional` (IMDSv1 + IMDSv2 both enabled, v2 not enforced). **Amazon Linux 2023** sets `imds-support=v2.0`, so instances default to `required` (IMDSv2 only, IMDSv1 disabled) with hop limit 2

#### IMDS cheat sheet

| Keyword in the question                                         | Answer                                                                                                                   |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| The metadata service URL                                        | **`http://169.254.169.254/latest/meta-data/`**                                                                           |
| "Code on the instance needs its public IPv4"                    | **IMDS** — `meta-data/public-ipv4`                                                                                       |
| "App on EC2 needs to call S3 without storing access keys"       | **IAM role on the instance + SDK reads creds from IMDS** at `iam/security-credentials/<role>`                            |
| "Bootstrap script needs to read the user-data passed at launch" | `http://169.254.169.254/latest/user-data/`                                                                               |
| "Pulled credentials from the metadata service via SSRF"         | The **Capital One** pattern — IMDSv1 vulnerability; fix is **enforce IMDSv2**                                            |
| "Mitigate SSRF against the metadata service"                    | **`HttpTokens=required`** (forces IMDSv2 session-token handshake)                                                        |
| "Block all metadata access from inside the OS"                  | Local OS firewall rule blocking `169.254.169.254` — per-instance, high-overhead; IMDSv2 is the real answer               |
| "Metadata service uses TLS"                                     | ❌ No — IMDS is **plain HTTP** over link-local. Confidentiality comes from the local-only addressing, not from encryption |
| "169.254.169.254 reachable from outside the instance"           | ❌ No — link-local; never leaves the instance                                                                             |
| "Same IP address for IMDS in `us-east-1` and `ap-southeast-2`"  | ✅ Yes — link-local, identical on every instance everywhere                                                               |
| "Which Amazon Linux requires IMDSv2 by default"                 | **AL2023** (`imds-support=v2.0`, IMDSv1 disabled). **AL2** defaults to `optional` (both enabled)                         |

> [!tip] Exam Tip
> Pattern-match: *"discover own public IPv4 from inside the instance"* → **IMDS** at `meta-data/public-ipv4`. *"SDK on EC2 obtains credentials"* → **IAM role + IMDS** (no static keys). *"SSRF stole the role credentials from metadata"* → **enforce IMDSv2** (`HttpTokens=required`). *"the metadata endpoint URL"* → **`http://169.254.169.254/latest/meta-data/`** — memorize verbatim.

> [!warning] Common IMDS Gotchas
>
> - **IMDSv1 is unauthenticated** — any local process or SSRF-vulnerable web app reads everything including the **role's temporary credentials**. Treat the role's policy as production-secret-equivalent
> - **IMDS is plain HTTP**, not HTTPS. The link-local addressing is the only confidentiality boundary
> - **IMDS works without internet access** — link-local addressing routes inside the hypervisor; an instance in a fully private subnet still reads metadata fine
> - **Stop/Start changes the public IPv4** — and IMDS will return the new one immediately. Apps that cache `public-ipv4` need to re-read after a host migration
> - **OS firewall blocking 169.254.169.254 breaks SDK credential resolution** — the SDK has no other path to the role's temporary keys. Don't blanket-block; enforce IMDSv2 instead

### AMI — Amazon Machine Image

See [EC2 — AMI](../services/ec2.md#ami--amazon-machine-image) for full details.

- **AMI = a logical container, not a data carrier.** Holds an AMI ID, permissions, and a **block device mapping** pointing at EBS snapshots. The volume data lives in the snapshots — the AMI itself stores no bytes
- **Three sources**: **AWS-provided** (Amazon Linux, Windows base), **Community** (Red Hat, CentOS, Ubuntu), **Marketplace** (commercial software bundled). Marketplace billing = **instance hourly rate + per-hour software-license charge**
- **Three permission modes**: **Private** (default — owner-only) / **Public** (everyone) / **Explicit** (specific named AWS account IDs)
- **Regional** — one AMI lives in exactly one region. Same OS has a **different AMI ID per region**. **Deployable to any AZ within the region** (snapshots live on regional S3)
- **Four-phase lifecycle**: (1) Launch from an AMI, (2) Configure the instance, (3) Create Image (captures snapshots and writes the block device mapping), (4) Launch new instances from the new AMI. The flow runs in **both directions** — using an AMI to launch an instance, and capturing an instance as an AMI
- **AMIs cannot be edited.** To change an AMI, launch from it, modify the instance, capture a **brand-new** AMI
- **AMIs can be copied across regions** with `aws ec2 copy-image`. Copies the underlying snapshots; copied AMIs **default back to Private** — sharing does not travel with the copy
- **Encrypted AMIs** require [KMS](../services/kms.md) grants for services like Auto Scaling to launch from them. Cross-region copy re-wraps the snapshot DEK under a destination-region KMS key
- **Billing**: pay for the **EBS snapshot used data** the AMI references, not for the AMI container, and not for the source volume's full allocated size. A mostly-empty large volume produces a small, cheap snapshot
- **AMI baking** = configure an instance, install all software, capture as AMI, launch many identical instances. The canonical pattern for fleet provisioning and golden-image pipelines (EC2 Image Builder, Packer)

#### AMI cheat sheet

| Keyword in the question                          | Answer                                                                                                                                    |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| "Same OS, different AMI IDs across regions"      | Yes — AMIs are **regional**; same OS gets a distinct AMI ID in each region                                                                |
| "Move an AMI to another region"                  | **`aws ec2 copy-image`** — copies the underlying snapshots; permissions reset to Private                                                  |
| "Move an AMI to another AZ"                      | **No action needed** — an AMI deploys to any AZ in its home region                                                                        |
| "Modify an existing AMI"                         | ❌ Not possible — launch from it, configure, capture a **new** AMI                                                                         |
| "Share AMI with a specific AWS account"          | **Explicit** permission with that account's ID                                                                                            |
| "Share AMI with everyone"                        | **Public** permission — also the model for AWS-provided base images                                                                       |
| "Default AMI permission"                         | **Private** (owner-only)                                                                                                                  |
| "Marketplace AMI cost model"                     | Normal instance hourly rate **plus** a per-hour software-license charge                                                                   |
| "What does the AMI actually store?"              | AMI ID + permissions + **block device mapping** pointing at EBS snapshots. **No volume bytes inside the AMI**                             |
| "Block device mapping"                           | A table: snapshot ID + original device ID (e.g. `/dev/xvda` for boot, `/dev/xvdf` for data). Used at launch to recreate the volume layout |
| "How much does an idle AMI cost?"                | **Snapshot used data × $/GB-month** — not the volume's allocated size, not the AMI itself                                                 |
| "Configure once, launch many"                    | **AMI baking** — capture the configured instance as an AMI                                                                                |
| "AMIs created from one AZ"                       | Deployable to **all AZs in the region** — the AMI's snapshots are stored on S3, which is regional                                         |
| "Encrypted AMI fails to launch via Auto Scaling" | Auto Scaling needs a **KMS grant** on the AMI's encryption key                                                                            |

> [!tip] Exam Tip
> Pattern-match: *"create many identical instances from a customized server"* → **AMI baking** (capture as AMI, then launch). *"AMI not available in region X"* → **copy AMI** to that region first. *"edit an AMI to update the package version"* → ❌ not possible; (1) launch from the AMI, (2) modify the instance, (3) capture a new AMI. *"AMI created in AZ-a — can it launch in AZ-b?"* → ✅ yes, same region is enough. *"Marketplace AMI total cost"* → instance rate **plus** per-hour license. *"Who owns the data in an AMI?"* → the EBS snapshots underneath; the AMI is just a pointer.

> [!warning] Common AMI Gotchas
>
> - **AMIs are immutable.** Every change is a new AMI. The exam trap framing is *"edit the AMI to upgrade the OS"* — always wrong
> - **Region-locked.** Multi-region launch requires `copy-image` first. Permissions reset to Private after copy
> - **Logical container, not data.** Deleting the underlying snapshots breaks the AMI; AMI lifecycle and snapshot lifecycle must be managed together
> - **Snapshot used-data billing.** A 100 GB volume with 5 GB written produces an AMI billed for ~5 GB, not 100 GB. The cost is in the *snapshot*, not the AMI object
> - **Marketplace AMIs carry a license charge** bundled into the per-hour bill — not just the instance rate
> - **Encrypted AMI + Auto Scaling** silently fails without a KMS **grant** on the encryption key — common cross-account / multi-region trap

### EC2 Bootstrapping — User Data & cfn-init

See [EC2 — Bootstrapping](../services/ec2.md#bootstrapping), [CloudFormation — cfn-init](../services/cloudformation.md), and [Boot Time to Service Time](../patterns/boot-time-optimization.md) for full details.

- **Bootstrapping** = self-configure at launch. In EC2 it's delivered via **user data**, read from [IMDS](../services/ec2.md#instance-metadata-service-imds) at **`http://169.254.169.254/latest/user-data/`** (same service as metadata, different path)
- User data **runs once, at first launch only**, **as root**, and is **opaque to EC2** (not validated/interpreted)
- **Edit + restart does NOT re-run user data** — the new data is visible but never re-executed. One-shot
- **Failed bootstrap does not fail the instance** — it stays running and passes **2/2 status checks** → silent **bad config**. EC2 can't tell good config from bad
- **Not a secret store** — readable by anyone with code/shell on the box; **enforcing IMDSv2** (default on Amazon Linux 2023) closes the *remote SSRF* path but does **not** make user data secret. **No long-term credentials** — use an [IAM role](../services/iam.md#iam-roles)
- **16 KB limit** — for bigger payloads, bootstrap a small script that **downloads** the rest
- **Base64**: console **auto-encodes**; **CloudFormation requires `Fn::Base64`**
- Diagnose with **`/var/log/cloud-init-output.log`** (user data) and **`/var/log/cfn-init-cmd.log`** (cfn-init)
- **cfn-init** = desired-state config management (version-aware packages/files/services/commands), defined in **`AWS::CloudFormation::Init`** metadata, **invoked from user data**. Procedural *or* desired-state, vs user data's procedural-only
- **cfn-hup** lets cfn-init **re-run on stack-metadata updates** — user data can't (it's one-shot)
- **CreationPolicy + cfn-signal** (`cfn-signal -e $?`) is the only way CloudFormation **knows** in-instance bootstrapping succeeded: it **holds** the resource at a non-complete state and **waits for a signal** within a **timeout** (e.g. `PT15M`); no signal = assumed failure. Used on **EC2 instances** and **Auto Scaling Groups**
- **Boot time to service time** = provisioning time + post-launch config time. Three approaches: **bootstrap** (flexible, slow each launch), **AMI baking** (fast, frozen config), **combine both** (bake slow/stable part + bootstrap fast-changing part = optimal)

> [!tip] Exam Tip
> Trigger phrases: *"run a script at launch / configure on first boot"* → **user data**. *"failed bootstrap but instance still healthy / passes status checks"* → user data is **opaque**; detect failure with **cfn-signal**. *"re-apply config on stack update, not just first launch"* → **cfn-init + cfn-hup** (NOT user data). *"ensure a specific package version / complex desired-state bootstrapping"* → **cfn-init**. *"don't store credentials in user data"* → use an **IAM role**. *"minimize boot time to service time while keeping flexibility"* → **combine AMI baking + bootstrapping** (bake the 90%, bootstrap the 10%).

> [!warning] Common Bootstrapping Gotchas
>
> - **User data is one-shot.** Edit + stop/start does not re-run it. Re-runnable config = **cfn-init + cfn-hup**
> - **A broken bootstrap looks healthy.** Running + 2/2 status checks ≠ correctly configured. CloudFormation only knows via **CreationPolicy + cfn-signal**
> - **Console auto-base64s, CloudFormation does not** — forget `Fn::Base64` in a template and the launch breaks
> - **`-e $?`** in cfn-signal forwards the **exit code of the preceding cfn-init run** — that's what becomes the success/failure signal
> - **User data is not encrypted/secret** — IMDSv2 stops remote SSRF but a shell user still reads it; never put long-term secrets there

### EC2 Instance Roles & Instance Profiles

See [EC2 — Instance Roles & Instance Profiles](../services/ec2.md#instance-roles--instance-profiles) and [IAM Roles](../services/iam.md#iam-roles) for full details.

- **EC2 instance role** = an [IAM role](../services/iam.md#iam-roles) whose **trust policy trusts `ec2.amazonaws.com`**. It's how an app on EC2 reaches other AWS services with **no long-term access keys on the instance**
- **Instance profile** = a wrapper around **one** IAM role. A role **cannot attach to an instance directly** — the **profile is what attaches**
- **Console**: creating the role **auto-creates a same-name instance profile** (the two-object reality is hidden). **CLI / CloudFormation / API**: you create **role and profile separately**, and they **can have different names** (`AWS::IAM::Role` + `AWS::IAM::InstanceProfile` are two resources)
- **Credential delivery**: temporary creds (Access Key + Secret + session token + expiry) are delivered via **[IMDS](../services/ec2.md#instance-metadata-service-imds)** at **`iam/security-credentials/<role-name>`**, and **EC2 + STS auto-renew them before expiry** → always valid while attached. Apps should re-read metadata, not over-cache
- **Credential provider chain**: instance profile credentials are the **last** source the AWS CLI checks → an instance with a role needs **zero credentials configured**. But **anything higher silently shadows the role** (env vars `AWS_ACCESS_KEY_ID`, `~/.aws/credentials`, CLI flags)

> [!tip] Exam Tip
> *"App on EC2 needs to access S3 / [DynamoDB](../services/dynamodb.md) / etc."* → **instance role** (not access keys). *"Access keys / hardcoded credentials on an instance"* → **bad practice**, replace with an instance role. *"What is attached to the instance — role or profile?"* → the **instance profile**. *"How do role credentials reach the app without keys?"* → **temporary creds via IMDS, auto-renewed by EC2 + STS**. *"Role attached but app uses wrong/expired permissions"* → **higher-precedence credentials (env vars / credentials file) shadowing the role** — remove them so the chain falls through to IMDS. Mnemonic **RPM**: **R**ole holds permissions, **P**rofile attaches, **M**etadata delivers.

### EC2 Launch Configurations & Launch Templates

See [EC2 — Launch Configurations & Launch Templates](../services/ec2.md#launch-configurations--launch-templates) for full details.

- Both are **immutable documents** that pre-define instance config ([AMI](../services/ec2.md#ami--amazon-machine-image), type, storage, key pair, networking, [user data](../services/ec2.md#bootstrapping), [IAM role](../services/ec2.md#instance-roles--instance-profiles)) — **neither can be edited**: create a **new LC** or a **new LT version**
- **LC** = older, **no versioning**, **only consumer is an Auto Scaling Group** — and **deprecated**: accounts created on or after **Oct 1, 2024** cannot create LCs by any method
- **LT** = **superset**: versions, **direct instance launch from console/CLI**, newer features (**T2/T3 Unlimited, placement groups, Capacity Reservations**), and **Mixed Instances Policy** (multiple instance types + Spot and On-Demand in one ASG)

> [!warning] Gotcha
> *"How do I update a launch configuration?"* — you **can't**; you create a new one and repoint the ASG. An LT isn't edited in place either — you create a **new version**. Any answer implying an in-place edit is a trap.

#### LC vs LT trigger-keyword table

| Exam trigger                                                           | Points to                                                |
| ---------------------------------------------------------------------- | -------------------------------------------------------- |
| "versions", "default/latest version"                                   | **Launch Template**                                      |
| "launch instances directly from console/CLI"                           | **Launch Template**                                      |
| "mixed instance types", "Spot + On-Demand in one ASG"                  | **Launch Template** (Mixed Instances Policy)             |
| "T2/T3 Unlimited", "placement groups", "Capacity Reservations"         | **Launch Template**                                      |
| "newest instance types" / "recommended" / "modern approach"            | **Launch Template**                                      |
| "only used by Auto Scaling Groups", "no versioning", "older mechanism" | **Launch Configuration**                                 |
| "edit the existing launch config/template"                             | **Trap** — neither is editable; new LC or new LT version |

> [!tip] Exam Tip
> If a question asks for the **recommended** or **modern** approach, or mentions **new instance types, mixed/Spot purchasing, or versioning**, the answer is **launch templates** — launch configurations are legacy and formally deprecated.

### EC2 Placement Groups

See [EC2 — Placement Groups](../services/ec2.md#placement-groups) for full details.

- Placement groups influence the **physical placement** of instances on EC2 hosts: **Cluster** packs them close together for performance; **Spread** and **Partition** separate them for resilience
- **Cluster**: same rack, often same host; **10 Gbps single-stream** throughput (normal cap is 5 Gbps); lowest latency and max PPS in AWS; **single AZ only** (locked to the AZ of the first instance launched, not chosen at creation); little to no resilience
- **Spread**: each instance on its own isolated rack with its own network and power; **hard limit of 7 running instances per AZ *per group*** (per placement group, **not** account-wide — for more, AWS recommends multiple spread groups, but there's no guarantee of spread *between* groups); can span multiple AZs
- **Partition**: **max 7 partitions per AZ**, each on isolated racks with no sharing between partitions; **unlimited instances**; you pick the partition or EC2 decides; exposes partition topology to applications

#### Placement group decision table

| If the question emphasises…                                                                              | Pick          |
| -------------------------------------------------------------------------------------------------------- | ------------- |
| Highest performance, lowest latency, fastest node-to-node (HPC, scientific)                              | **Cluster**   |
| Small number of critical instances, max availability, separate blast radius                              | **Spread**    |
| Large-scale distributed/replicated app, more than 7 instances, topology-aware (HDFS / HBase / Cassandra) | **Partition** |

#### Numbers to memorise

| Fact                              | Value                                             |
| --------------------------------- | ------------------------------------------------- |
| Cluster single-stream throughput  | **10 Gbps** (vs **5 Gbps** normal)                |
| Spread instances per AZ           | **7** (hard limit, **per group** not per account) |
| Partition *partitions* per AZ     | **7** (max)                                       |
| Partition instances per partition | Unlimited (your choice)                           |

> [!tip] Exam Tip
> The cluster performance numbers are conditional: you need a **supported instance type** with high-performance networking **and enhanced networking enabled** on all members. Same instance type and launching all members at the same time are **recommended, not mandatory**. And the mnemonic: **C**luster is **C**lose together for **C**ompute performance; **S**pread is **S**eparated, **S**even per AZ; **P**artition gives 7 **P**artitions per AZ for **P**arallel-processing apps at scale.

> [!warning] Common Placement Group Gotchas
>
> - **Cluster cannot span AZs** — it can span **VPC peers**, but that significantly degrades performance
> - **Cluster has low or no resilience** — co-location means one hardware failure can take out the whole group
> - **Spread caps at 7 running instances per AZ *per group*** (not account-wide) — a scenario needing more separated instances is pointing at **Partition**, not Spread (the most common trap: separation **and** lots of instances means Partition). You *can* run multiple spread groups for more than 7, but AWS gives **no spread guarantee between** groups — so it's not a substitute for Partition at scale
> - **Spread does not support Dedicated Instances or Dedicated Hosts**
> - **Partition isolation is per partition, not per instance** — 10 instances in 1 partition all share that partition's fate

### EC2 Enhanced Networking

See [EC2 — Enhanced Networking](../services/ec2.md#enhanced-networking) for full details.

- **What it is:** a feature that boosts EC2 network performance via **SR-IOV** (Single Root I/O Virtualization), which makes the host's **physical NIC virtualization-aware**
- **How it works:** instead of one shared physical card the host arbitrates **in software**, the card presents **per-instance logical cards**. Each instance gets exclusive access to its own, and the **host CPU is taken out of the packet path** (offloaded to hardware, the **ENA** on AWS)
- **What it delivers:** more bandwidth, **higher packets per second (PPS)**, and — the key phrase — **consistent low latency**
- **Availability:** enabled by default or free on **most modern instance types**; SAA does not test the enablement mechanics
- **Why it matters:** it's the **prerequisite for [cluster placement groups](../services/ec2.md#cluster-placement-groups)** — the 10 Gbps / low-latency cluster numbers require it on every member

> [!tip] Exam Tip
> Map these triggers straight to **enhanced networking**: **"consistent low latency," "high packets per second / PPS," "small packets over a small isolated network,"** and **"required for cluster placement groups."** Don't overthink it — the SAA exam cares about *what it gives you* (PPS + consistent latency) and *what depends on it* (cluster placement), not how to turn it on.

### EC2 EBS-Optimized Instances

See [EC2 — EBS-Optimized Instances](../services/ec2.md#ebs-optimized-instances) for full details.

- **What it is:** a per-instance **on/off** flag that gives **[EBS](../services/ebs.md) traffic a dedicated network lane**, separate from regular data networking
- **Problem it solves:** historically one **shared network stack** carried both data and EBS storage traffic, so the two **contended** and capped each other's performance. EBS is **network-attached**, so its I/O always crosses the network
- **Effect:** faster EBS throughput / IOPS, and storage vs data traffic **no longer compete**
- **Required for:** instances driving **high throughput / IOPS** volumes (e.g. **gp2 / io1**); it's the instance-side half of the [EBS per-instance ceiling](../services/ebs.md#the-two-ceiling-rule)
- **Availability:** **default-on and free on most modern instance types** (disabling does nothing); paid opt-in on some older types

> [!tip] Exam Tip
> Two distinct "dedicated capacity" features that are easy to confuse: **enhanced networking** dedicates the **data**-network path (PPS, consistent latency, cluster placement); **EBS optimization** dedicates the **storage**-network path (EBS throughput/IOPS, no contention with data traffic). If the scenario is about **disk performance or EBS/network contention**, it's **EBS-optimized**; if it's about **packet rate / node-to-node latency**, it's **enhanced networking**.

### EBS Service Architecture

See [EBS — Service Architecture](../services/ebs.md#service-architecture) for full details.

- **EBS is AZ-scoped.** A volume lives in exactly one AZ; an instance must be in the same AZ to attach it.
- **File system comes from the OS.** EBS provides the raw block device; the EC2 instance's OS (Linux/Windows) formats and mounts it. EBS has no concept of EXT4 or NTFS — that's entirely the instance's responsibility.
- **Snapshots are stored in S3** (AWS-managed, regionally resilient — 3+ AZs). They are the only way to move volume data across AZs or regions.
- **Cross-region DR path:** snapshot the volume → copy snapshot to target region → restore volume there.
- **Multi-attach conditions (all three must hold):** (1) supported volume type, (2) same AZ for all instances, (3) cluster-aware application that coordinates writes.

> [!tip] Exam Tip
> Pattern-match: *"instance needs access to a volume in another AZ"* → **snapshot → restore in target AZ** (direct cross-AZ attachment is impossible). *"cross-region disaster recovery for EBS data"* → **snapshot → copy snapshot to target region → create volume from copy**. *"two instances share an EBS volume"* → **multi-attach**, but only with supported volume types, same AZ, and cluster-aware software. *"where does the EXT4 / NTFS file system on an EBS volume come from?"* → **the EC2 instance's OS** — EBS only provides the raw block device.

> [!warning] Gotcha — EBS replication is within an AZ only
> EBS replicates data across multiple physical disks *inside* the AZ — protecting against single-disk failure. It does **not** protect against an AZ-wide failure. The only protection against AZ failure is to snapshot the volume (which lands in S3 across 3+ AZs) and be prepared to restore in a different AZ.

### EBS Snapshots

See [EBS — Snapshots](../services/ebs.md#snapshots--crossing-the-az-and-region-boundaries) for full details.

- **First snapshot = full copy of used data only** (not allocated size). A 40 GB volume with 10 GB written → first snapshot bills for 10 GB.
- **Subsequent snapshots = incremental** — only changed/new data since the previous snapshot.
- **Incrementals are self-sufficient** — deleting any snapshot is safe; AWS relocates referenced data so later snapshots remain restorable.
- **Billing: GB-month on used data**, not allocated size. Frequent snapshots stay cheap because only deltas accumulate.
- **Restore is lazy** — blocks pulled from S3 on first access; unrestored blocks read at lower performance until warmed.
- **`dd`**: reads every block to force EBS to pull all data from S3. Free, manual warm-up.
- **FSR (Fast Snapshot Restore)**: makes restores instant. Extra cost. **Limit: 50 sets per region**, counted **per snapshot + AZ combination** (one snapshot × 4 AZs = 4 sets used).

> [!tip] Exam Tip
> Pattern-match: *"first snapshot size"* → **used data, not allocated volume size**. *"delete an incremental snapshot"* → **safe** — AWS preserves later snapshots automatically. *"slow I/O right after restoring from a snapshot"* → **lazy restore** — pre-warm with `dd` (free) or enable FSR (instant, costs extra). *"FSR limit"* → **50 per region**, counted per **snapshot + AZ**. *"billing for frequent snapshots"* → cheap — only deltas are stored; aggressive schedules don't scale cost the way you'd expect.

> [!warning] Common Snapshot Gotchas
>
> - **Volume bills on allocated size; snapshot bills on used data.** Same 40 GB volume, two different billing bases.
> - **FSR limit is per snapshot + AZ.** 1 snapshot × 4 AZs = 4 FSR sets consumed from your 50-set budget.
> - **Restore creates a new volume — not an in-place recovery.** The original volume and the restored copy coexist until you delete one.
> - **Snapshot copy is needed before cross-region restore.** You cannot restore a snapshot directly into a different region — copy it first.

### EBS Volume Types — GP2 vs GP3

See [EBS — Elastic Block Store](../services/ebs.md) for full details.

**The single biggest conceptual difference:** GP2 is credit-based (performance depends on volume size); GP3 is flat-rate (same baseline regardless of size).

**GP2 key numbers:**

- Baseline: **3 IOPS/GB**, minimum floor **100 IOPS**, ceiling **16,000 IOPS**
- Crossover points: **33.33 GB** (formula beats floor) → **1 TB** (baseline = burst; credit system irrelevant) → **5.33 TB** (max IOPS reached)
- Burst: **3,000 IOPS** via a **5.4 million credit bucket** — new volumes start full (30 min @ 3,000 IOPS)
- Max throughput: **250 MB/s**

**GP3 key numbers:**

- Standard baseline (every volume, every size): **3,000 IOPS + 125 MB/s** — no formula, no bucket
- Provisioned extras (extra cost, independently configurable): up to **16,000 IOPS** + up to **1,000 MB/s**
- Base price: **~20% cheaper than GP2**
- **4× the max throughput** of GP2

**Migration:** use **Elastic Volumes** to switch GP2 ↔ GP3 with no downtime — but after moving to GP3, explicitly provision the IOPS/throughput you need above the 3,000/125 MB/s standard; GP3 does not auto-scale with volume size.

> [!tip] Exam Tip
> Pattern-match: *"small EBS volume has inconsistent, degrading IOPS over time"* → **GP2 credit exhaustion** — switch to GP3 or size the volume larger for a higher baseline. *"need to set IOPS independently of volume size"* → **GP3** (or Provisioned IOPS). *"cost-optimise existing GP2 volumes"* → migrate to GP3 (~20% cheaper base). *"4× throughput compared to GP2"* → GP3 max is 1,000 MB/s vs GP2's 250 MB/s. *"boot volume / general workload / no special requirement"* → GP3 (current AWS default).

> [!warning] Gotcha — GP2 credit trap
> A workload that **consistently exceeds the GP2 baseline** (e.g. 3,000 IOPS on a 100 GB volume whose baseline is only 300 IOPS) will eventually drain the credit bucket to zero. Once empty, performance collapses to the 300 IOPS baseline — not the 3,000 burst rate — and stays there until credits rebuild. Monitor `BurstBalance` CloudWatch metric for GP2 volumes.

> [!warning] Gotcha — GP2 → GP3 migration
> After migrating from GP2 to GP3, you lose the automatic size-based IOPS scaling. A 1 TB GP2 volume had a 3,000 IOPS baseline by formula; a 1 TB GP3 volume has a 3,000 IOPS baseline by standard — but a 2 TB GP3 volume is still only 3,000 IOPS unless you explicitly provision more. Always confirm IOPS requirements before migrating large volumes.

### EBS Volume Types — Provisioned IOPS SSD (IO1, IO2, IO2 Block Express)

See [EBS — Provisioned IOPS SSD](../services/ebs.md#provisioned-iops-ssd--io1-io2-io2-block-express) for full details.

The Provisioned IOPS SSD family is for workloads that need **extreme, consistent, low-jitter performance** — I/O-intensive relational and NoSQL databases where average latency isn't enough; you need it *predictable*.

**Key numbers to know:**

| Attribute                 | IO1          | IO2          | IO2 Block Express |
| ------------------------- | ------------ | ------------ | ----------------- |
| Max IOPS per volume       | 64,000       | 64,000       | **256,000**       |
| Max throughput per volume | 1,000 MB/s   | 1,000 MB/s   | **4,000 MB/s**    |
| Max IOPS/GB ratio         | 50           | 500          | 1,000             |
| Per-instance max IOPS     | 260,000      | 160,000      | 260,000           |
| Size range                | 4 GB – 16 TB | 4 GB – 16 TB | 4 GB – **64 TB**  |

**The two-ceiling rule (most exam-tested concept):**

- **Per-volume ceiling**: what a single volume can deliver (IO1/IO2 = 64,000 IOPS).
- **Per-instance ceiling**: what can flow into one EC2 instance from all attached volumes combined (IO1 = 260,000 IOPS). These are separate ceilings — to saturate the per-instance ceiling you need multiple volumes in parallel.
- The per-instance ceiling is further capped by **EC2 instance type and size** — only the newest, largest instances reach these peaks. **All three must align: volume type + instance type + instance size.**

**IOPS/GB ratio caps:** IOPS are decoupled from size, but bounded by the ratio. Example: a 10 GB IO2 volume caps at 5,000 IOPS (500 × 10), not 64,000. To reach 64,000 IOPS you need at least 128 GB.

> [!tip] Exam Tip
> Pattern-match: *"need more than 16,000 IOPS"* → **IO1/IO2** (or Block Express). *"consistent, predictable sub-millisecond latency for a database"* → **IO1/IO2** (GP2/GP3 burst model doesn't guarantee consistency). *"high IOPS on a small volume"* → **IO2** (500 IOPS/GB ratio vs GP2's 3 IOPS/GB). *"per-instance performance ceiling"* → remember it exists separately from per-volume; multiple volumes needed to saturate it. *"256,000 IOPS / 64 TB volume"* → **IO2 Block Express**. *"extreme performance but I don't need persistence"* → [**Instance Store**](../services/ec2.md#storage-options) beats even Block Express, but data is ephemeral.

> [!warning] Gotcha — IO2 per-instance ceiling is lower than IO1
> IO2's per-instance ceiling is **160,000 IOPS** — lower than IO1's **260,000 IOPS**. This seems counterintuitive (IO2 is the newer type). AWS split the headroom: IO2 standard sits at 160k, while the premium capability went into Block Express (which matches IO1's 260k). Don't assume "newer = higher per-instance ceiling" for IO1 vs IO2 standard.

> [!warning] Gotcha — minimum size is 4 GB, not 1 GB
> GP2 and GP3 start at **1 GB**. IO1, IO2, and Block Express all start at **4 GB**. A question about a very small volume with extreme IOPS requirements needs both the ratio math and the 4 GB floor check.

### EBS Volume Types — HDD (ST1, SC1)

See [EBS — HDD-Based Volumes](../services/ebs.md#hdd-based-volumes--st1-and-sc1) for full details.

The HDD family is for **sequential, throughput-oriented workloads where cost matters more than IOPS**. HDDs have physical moving parts — appropriate only when access patterns play to their strength (large sequential I/O).

**Key numbers:**

| Attribute         | ST1 (Throughput Optimized)                | SC1 (Cold HDD)                     |
| ----------------- | ----------------------------------------- | ---------------------------------- |
| Relative cost     | Cheap                                     | **Cheapest EBS storage of all**    |
| Size range        | 125 GB – 16 TB                            | 125 GB – 16 TB                     |
| I/O block size    | **1 MB** (vs 16 KB on SSD)                | **1 MB**                           |
| Max IOPS          | 500                                       | 250                                |
| Max throughput    | 500 MB/s                                  | 250 MB/s                           |
| Baseline          | 40 MB/s per TB                            | 12 MB/s per TB                     |
| Burst             | 250 MB/s per TB                           | 80 MB/s per TB                     |
| Access pattern    | Frequent, sequential                      | Infrequent, cold, archival         |
| Typical workloads | Big data, data warehouses, log processing | Cold data, archives, few scans/day |

**1 MB I/O block rule:** on HDD, 1 IOPS = 1 MB — so the IOPS and MB/s ceilings are always the same number (500 IOPS = 500 MB/s). The credit bucket scales in **MB/s per TB** (not IOPS per GB like GP2).

> [!tip] Exam Tip
> Two HDD discriminators: (1) **random vs sequential** — the moment the question mentions IOPS, random access, databases, or low latency, HDD is ruled out. (2) **frequent vs infrequent** — ST1 for frequently accessed sequential data (big data, logs, warehouses); SC1 for cold/rarely accessed archival data. SC1 = the lowest-cost EBS storage overall.

> [!warning] Gotcha — 125 GB minimum and no boot volumes
> Both ST1 and SC1 have a **125 GB minimum** — much larger than GP2/GP3's 1 GB minimum. Neither can be used as a boot volume. A question describing a small volume, a boot volume, or a random-access workload cannot use HDD even if cost is the primary driver.

### EBS Encryption

See [EBS — Encryption](../services/ebs.md#ebs-encryption) for full details.

- **EBS encryption is envelope encryption built on [KMS](../services/kms.md)** — the KMS key wraps a per-volume DEK; the DEK does the bulk crypto on the EC2 host with **AES-256**. The KMS key itself **never encrypts volume data**
- **API used at volume create**: **`GenerateDataKeyWithoutPlaintext`** — EBS receives only the encrypted DEK and stores it in the volume's metadata. The plaintext DEK is fetched via a separate **`Decrypt`** call at first attach and lives **only in EC2 host memory**, never on disk
- **Transparent to the guest OS** — the file system sees plaintext bytes. If a question requires the **OS to hold the keys**, that's **software disk encryption** (LUKS / BitLocker), not EBS encryption
- **No cost for encryption itself, no measurable performance penalty** — enable by default. Only [KMS request charges](../services/kms.md) for the create-time and attach-time API calls
- **AWS-managed default key** `aws/ebs` works for most cases; a **[Customer Managed Key](../services/kms.md#key-ownership-and-management)** is required for custom key policy, rotation control, [grants](../services/kms.md#key-policy--grants), or cross-account sharing of encrypted snapshots
- **Account-level default encryption** is a per-region switch — turn it on in every region to enforce "no unencrypted EBS volumes"
- **Encryption is irreversible** — no console / CLI / API removes it. Going *from* encrypted *to* unencrypted requires an OS-level copy onto a separate unencrypted volume. Going the other direction (encrypting an existing unencrypted volume) uses **`CopySnapshot` with encryption enabled**

#### DEK reuse — the chain rule

| Resource                             | DEK                                   |
| ------------------------------------ | ------------------------------------- |
| Two volumes created **from scratch** | **Different** DEKs                    |
| Snapshot of an encrypted volume      | **Same DEK** as source                |
| Volume restored from that snapshot   | **Same DEK** as snapshot (and source) |

A single DEK threads through: **original volume → its snapshots → restored volumes**. Re-keying requires `CopySnapshot` with a different KMS key.

> [!tip] Exam Tip
> Pattern-match: *"how does KMS encrypt a 1 TB volume given the 4 KB ceiling"* → **it doesn't** — EBS does envelope encryption with a DEK; KMS only wraps the DEK. *"encrypt all new volumes in an account"* → **account-level default encryption** (per region). *"add encryption to an existing volume"* → snapshot → `CopySnapshot` with KMS key → restore. *"remove encryption"* → only via OS-level data copy onto an unencrypted volume — there is no API for it. *"OS must hold the keys / hypervisor not trusted"* → **software disk encryption** (LUKS/BitLocker) inside the guest, *not* EBS encryption. *"how many DEKs after N snapshots + M restored volumes of one source"* → **one**.

> [!warning] Gotcha — the KMS key does not encrypt the data
> Common exam confusion: the KMS key encrypts the **per-volume DEK**, not the volume contents. AES-256 happens on the EC2 host using the DEK. This is exactly why the [4 KB direct-encrypt limit](../services/kms.md#direct-encrypt--decrypt--4-kb) doesn't apply to multi-TB EBS volumes.

> [!warning] Gotcha — irreversibility
> Encryption is a **one-way door** like [versioning enable](../services/s3.md#state-machine--one-way-door) and [Object Lock](../services/s3.md#object-lock). Once on, no `DisableEncryption` API exists. Confirm KMS key choice and account-level defaults before creating production volumes.

> [!warning] Gotcha — KMS key region scope flows to EBS encryption
> KMS keys are **regional**. The account-level default-encryption setting and any customer-managed key referenced by EBS must exist in **every region** where you want encrypted volumes. Copying an encrypted snapshot to another region requires the destination region to have access to a KMS key (the source key's region won't decrypt it).

## EFS

See [EFS](../services/efs.md) for full details.

- **Network file system for Linux** — AWS's implementation of **NFS v4** (current docs: v4.0 and v4.1). Mounted by **many EC2 instances simultaneously, across AZs**; contrast with [EBS](../services/ebs.md) = block, one instance, one AZ
- **Linux only** — a question mentioning **Windows** shared file storage disqualifies EFS; the answer is **FSx for Windows File Server**
- Uses **POSIX permissions**; created **inside a [VPC](../services/vpc.md)**
- **Access is via mount targets**, never directly to the file system — mount targets live in **subnets** and take an **IP from the subnet's range**
- **HA rule:** a mount target in **every AZ the VPC uses** (same rule as NAT gateways). One mount target in one AZ is **not** highly available
- **Private by default** (VPC-isolated), but reachable from on-premises over **VPN / Direct Connect / VPC peering** if configured — don't assume EFS is unreachable from on-prem
- Enables **stateless EC2 instances** — media/files live outside the instance, supporting [scaling](../patterns/scaling.md) and self-healing

### EFS cheat sheet

| Attribute         | Course answer                                                                                           | Current AWS reality                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Performance modes | **General Purpose** *(default)* / **Max I/O** (parallel workloads, higher latency)                      | Max I/O is **previous-generation**; AWS recommends General Purpose for all file systems |
| Throughput modes  | **Bursting** *(default; scales with size, like GP2)* / **Provisioned** (size-independent, like IO1)     | A third mode, **Elastic** (auto-scales to demand), is now the **console default**       |
| Storage classes   | **Standard** *(default)* / **Infrequent Access**                                                        | Plus **One Zone / One Zone-IA** and **Archive**                                         |
| Cost optimization | **Lifecycle policies** move data between classes (like [S3](../services/s3.md#lifecycle-configuration)) | Same                                                                                    |

> [!tip] Exam Tip
> Pattern-match: *"shared file storage across multiple Linux instances"* means **EFS**. *"Latency-sensitive general file serving"* means **General Purpose**; *"highly parallel big-data / media processing"* means **Max I/O**. *"Throughput scales with stored data"* means **Bursting**; *"throughput independent of size"* means **Provisioned**. *"Automatically move rarely-used files to a cheaper tier"* means **lifecycle policy to IA**.

> [!warning] Gotcha — shifted defaults
> The course says the default throughput mode is **Bursting** and teaches 2 performance / 2 throughput / 2 storage-class options. Current AWS: default throughput is **Elastic**, Max I/O is previous-generation, and there are additional storage classes (One Zone, Archive). Older exam questions follow the course framing; watch the phrasing.

## RDS

See [RDS](../services/rds.md) for full details.

### Databases on EC2

- Running a DB on **[EC2](../services/ec2.md) is always architecturally worse** — the only question is whether a specific need outweighs it. It's **single-AZ by nature** (instance + EBS volume in one AZ), so it loses every HA/resilience scenario
- The **valid** reasons "DB on EC2" is the right exam answer: **engine or version AWS doesn't offer**, a **specific OS + DB version combo**, an **AWS-unsupported replication architecture**, or a genuine **OS/root-access** requirement (and even that is often met by **RDS Custom**)
- Same-AZ **private-IP** traffic is free; **cross-AZ** traffic is charged — relevant to split app/DB designs and Multi-AZ

### RDS core facts

- RDS = **"database server as a service"**, not "database as a service" — **one instance hosts multiple databases**. This phrasing is a deliberate exam trap
- **No SSH / no OS access** — exception: **RDS Custom**
- **Engines**: IBM Db2, MySQL, MariaDB, PostgreSQL, Oracle, SQL Server (Oracle and SQL Server carry licensing cost)
- RDS runs **inside a [VPC](../services/vpc.md)** (not a public-zone service); you **must pick a DB subnet group** at launch — a list of subnets RDS may use. Multi-AZ places primary + standby in **different AZs**, chosen at random unless specified
- **Each instance has its own dedicated [EBS](../services/ebs.md) storage** (primary and standby each separate) — relational DBs here use the **[ACID model](../concepts/db-transaction-model.md)**
- Billing is **resource-allocation** (like EC2), hourly rate **billed per second** — not usage-based. Snapshot storage **equal to DB storage is free**

### RDS Custom

Niche topic — **awareness only** (know it exists and what gap it fills). See [RDS — RDS Custom](../services/rds.md#rds-custom).

- **Middle ground** between standard RDS (fully managed, no OS access) and a **DB on [EC2](../services/ec2.md)** (fully self-managed): you keep RDS automation **and** get OS + engine access to customize
- **SQL Server and Oracle only** — the same engine pair that supports **TDE**. No MySQL / MariaDB / PostgreSQL
- **OS/engine access via SSH, RDP, and Session Manager** — the feature that separates it from standard RDS
- **Resources live in your account** — you **see** the EC2 instance, EBS volumes, and S3 backups (standard RDS hides these AWS-side and only injects **ENIs** into your VPC)
- To customize safely: **pause** database automation, make changes, then **resume** it

> [!tip] Exam Tip
> Pattern-match: *"managed RDS but I also need OS/engine access to customize"* → **RDS Custom**. *"which engines support RDS Custom?"* → **SQL Server + Oracle only** (same pair as TDE). *"I can see the EC2/EBS/S3 resources for my managed DB"* → RDS Custom (standard RDS hides them, ENI injection only).

### Multi-AZ vs Read Replicas

|                     | Standby (Multi-AZ)        | Read Replica                                  |
| ------------------- | ------------------------- | --------------------------------------------- |
| Replication         | **Synchronous**           | **Asynchronous**                              |
| Scope               | Same region, different AZ | Same- **or cross-region**                     |
| Purpose             | **HA / failover**         | **Read scaling + DR**                         |
| App-readable?       | **No**                    | **Yes** (own endpoint)                        |
| Automatic failover? | **Yes** (CNAME repoint)   | **No** — manual **promotion** only            |
| App changes needed? | No (same CNAME)           | **Yes** — point reads at the replica endpoint |

- **Count:** **5** read replicas per instance (the long-standing/course number) or **up to 15** in current AWS docs — know both. **Cascading** replicas (replicas of replicas) are allowed but **lag compounds** at each hop
- **Db2/Oracle exception:** RDS for Db2 standby replicas and RDS for Oracle mounted replicas don't accept user connections, so they are **DR-only** rather than read-scaling targets
- **RPO/RTO:** read replicas give **near-zero RPO and RTO** via fast **promotion** to a standalone read/write instance — but only for **failure recovery**. For **data corruption** the replica holds a copy of the corrupted data, so it's **no defence** (use point-in-time backups instead). Backups improve **RPO**; read replicas improve **RTO**
- Async means a small **replication lag** (seconds); cross-region replication is **AWS-managed and encrypted in transit**

> [!tip] Exam Tip
> Pattern-match: *"engine/version AWS doesn't support"* or *"OS/root access required"* → **DB on EC2** (one of the rare cases it's correct). *"automatic failover to an identical copy"* → **Multi-AZ standby (synchronous)**. *"offload read traffic"* or *"read scaling across regions"* → **read replica (asynchronous)**, except Db2 standby replicas and Oracle mounted replicas are DR-only. *"promote to a standalone instance"* / *"low RTO recovery"* → **read replica**, but *"recover from data corruption / deleted data"* → **NOT a replica** (it replicated the bad data) → **PITR backup**. *"app has no knowledge of the replica"* / *"separate endpoint"* → read replicas need **application changes**. *"multiple databases on one server"* → RDS (it's a server, not a single DB). *"backups without impacting the primary"* → Multi-AZ standby-sourced backups for MariaDB/MySQL/Oracle/PostgreSQL; **SQL Server still briefly suspends I/O**.

### Multi-AZ Instance vs Cluster

There are **two** Multi-AZ architectures — both **in-region HA only** (not DR), both **not** in the free tier:

|                   | Multi-AZ **Instance**                                   | Multi-AZ **Cluster**                                          |
| ----------------- | ------------------------------------------------------- | ------------------------------------------------------------- |
| Topology          | 1 primary + **1 standby** (2 AZs)                       | 1 writer + **2 readers** (3 AZs)                              |
| Standby readable? | **No** — passive failover target                        | **Yes** — readers serve reads                                 |
| Read scaling?     | No (need a read replica)                                | **Yes** via the reader endpoint                               |
| Replication       | **Synchronous**, storage-level                          | **Semi-synchronous** quorum (≥ 1 reader ack), transaction-log |
| Failover          | **60–120 s** (DNS repoint)                              | **~35 s** floor / **25–75 s** typical (no crash recovery)     |
| Storage / HW      | EBS, standard                                           | **Local NVMe SSD flushed to EBS**, **Graviton**               |
| Engines           | Db2, MariaDB, MySQL, Oracle, PostgreSQL, **SQL Server** | **MySQL & PostgreSQL only**                                   |
| Endpoints         | Single **database CNAME**                               | **Cluster** (writer) / **Reader** / **Instance** (test-only)  |

- Instance-mode failover is a **DNS repoint**, not a data-recovery op — the standby already holds a synchronous copy, so the 60–120 s is DNS propagation + client caching. **Disable app-side DNS caching** to shorten it
- Cluster-mode replication is **semi-synchronous** in AWS docs (commit on **≥ 1 of 2** standbys), even though some material says "synchronous" — know both framings
- Cluster readers behave like **read replicas** (serve reads, exhibit per-reader replica lag) but replicate **semi-synchronously** rather than asynchronously — so a cluster delivers read scaling **and** a quorum-backed commit in one HA deployment, where a Multi-AZ *instance* would need separate async read replicas bolted on
- Cluster's **Instance endpoint** points at one specific instance and **does not fail over** — testing/fault-finding only, never production

> [!tip] Exam Tip
> Trigger words: *"readable standby"* / *"faster failover"* / *"lower commit (write) latency"* → **Multi-AZ cluster**. *"SQL Server or Oracle + Multi-AZ"* → must be **instance** mode (cluster is MySQL/PostgreSQL only). *"read scaling"* → **cluster or read replica, never instance mode**. *"Multi-AZ for cross-region / DR"* → wrong — Multi-AZ is in-region; use a **cross-region read replica**.

> [!warning] Gotcha — Aurora ≠ RDS
> **[Amazon Aurora](../services/aurora.md)** is a separate AWS-built engine with a **shared-cluster storage** architecture — *not* the dedicated-EBS-per-instance model RDS uses (cluster mode still keeps **per-instance local storage**, and caps at exactly **2 readers** where Aurora allows more). Don't carry RDS storage facts over to Aurora questions. See the [Aurora section](#aurora) below.

### Backups, Snapshots & Restore

Two mechanisms, both stored in **AWS-managed [S3](../services/s3.md)** (regionally resilient; **invisible in your own S3 console** — exam trap):

|              | Manual snapshot                                                        | Automated backup                                                   |
| ------------ | ---------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Trigger      | **You** (or a script)                                                  | **RDS, once/day** in a backup window                               |
| Extra stream | None                                                                   | **Transaction logs every 5 min**                                   |
| Retention    | **Never expires** — manual delete only; **survives instance deletion** | **0–35 days** (0 = disabled)                                       |
| PITR         | ❌ restores only to its **fixed moment**                                | ✅ **~5-min Recovery Point Objective (RPO)** point-in-time recovery |
| Scope        | **Instance-level** (all DBs)                                           | Instance-level                                                     |

- **First snapshot is full, the rest incremental** — true for **both** manual snapshots **and** automated backups (AWS treats an automated backup as an automated snapshot)
- **PITR works** by restoring the closest snapshot then **replaying transaction logs** to the chosen time — so it requires automated backups; a manual snapshot alone can't do it
- **Keep data past the 35-day max on deletion** → take a **final manual snapshot**. "Retain automated backups on delete" **still expires** them
- **Cross-region backup replication** (snapshots + logs) is **opt-in, not default**, and costs extra
- A **restore always creates a new instance with a new endpoint** — re-point apps; it never overwrites the source. Restore **time is your Recovery Time Objective (RTO)** (slow for large DBs); backups improve **RPO**, **read replicas** improve **RTO**
- On **Multi-AZ instance** mode, MariaDB/MySQL/Oracle/PostgreSQL backups run from the **standby** (no primary I/O suspension); **SQL Server still briefly suspends I/O**. On **Single-AZ** the sole instance takes a brief I/O pause

> [!tip] Exam Tip
> Pattern-match: *"backups not in my S3 buckets"* → **AWS-managed, RDS console only**. *"restore to the exact moment before failure"* → **automated-backup PITR**, not a manual snapshot. *"keep the data after deleting the instance / past 35 days"* → **final manual snapshot**. *"first backup full, then incremental"* → **both** snapshot types. *"5 minutes"* → **transaction-log interval** / achievable PITR RPO. *"replicate backups to another region"* → opt-in, extra cost. *"restore is slow / improve recovery time (RTO)"* → that's the **restore window**; use a **read replica**, not bigger/faster backups.

### Data Security

Four separate concerns — keep **authentication vs authorization** and **in-transit vs at-rest** apart (see [RDS — Data Security](../services/rds.md#data-security)):

- **Encryption in transit** — **SSL/TLS**, all engines, can be made **mandatory per user**. Trigger: *"force/require SSL"*
- **Encryption at rest, default** — **KMS + EBS**, done **by the host** below the engine (the engine writes plaintext, unaware). A **KMS key** (customer- or AWS-managed `aws/rds`) generates **DEKs** ([envelope encryption](../services/kms.md), **AES-256**). Storage, logs, snapshots, and replicas all use the **same key**
- **Encryption at rest, engine-native** — **TDE**, **SQL Server and Oracle only**. **MySQL / MariaDB / PostgreSQL cannot use TDE** (they still get at-rest protection via the host-side KMS/EBS path)
- **No AWS in the chain of trust** — **RDS Oracle + TDE + CloudHSM** (you hold the keys; AWS never sees them). Oracle-specific
- **IAM authentication** — log in with a **15-minute token** from **`GenerateDBAuthToken`** instead of a DB password. **Authentication only — never authorization**: once in, the **local DB user** still governs permissions. Must be explicitly enabled; **TLS is mandatory** (token is sent as a cleartext password); mint a **fresh token per new connection**

| At-rest mechanism       | Who encrypts                   | Engines                      |
| ----------------------- | ------------------------------ | ---------------------------- |
| **KMS + EBS** (default) | **The host**, below the engine | **All** engines              |
| **TDE**                 | **The DB engine** itself       | **SQL Server & Oracle only** |
| **TDE + CloudHSM**      | The engine, with **your** keys | **Oracle only**              |

> [!tip] Exam Tip
> Pattern-match: *"force/require SSL"* → **per-user mandatory TLS**. *"default at-rest, host does the encryption"* → **KMS + EBS** (KMS key → DEKs, AES-256). *"engine-native encryption"* → **TDE** (SQL Server/Oracle only — never MySQL/MariaDB/PostgreSQL). *"AWS must have no access to the keys"* → **Oracle + TDE + CloudHSM**. *"log in without a stored DB password using an AWS identity"* → **IAM authentication** (`GenerateDBAuthToken`, 15-min token). *"does IAM decide what the user can do in the DB?"* → **No, the local DB user does**.

> [!warning] Gotchas
>
> - **IAM auth ≠ authorization** — it only authenticates; the local DB user still controls permissions. The single most-tested point here
> - **TDE is restricted to SQL Server and Oracle** — "MySQL is encrypted" does **not** mean MySQL supports TDE (it uses host-side KMS/EBS instead)
> - **Encryption can't be removed, and can't be added in place** after creation — only via **snapshot → copy-with-encryption → restore**; a read replica's encryption state must **match its source**
> - **CloudHSM "no AWS trust" option is RDS Oracle-specific** — don't generalize it to other engines
> - **Don't confuse the KMS key (CMK) with the DEK** — the KMS key *generates* DEKs; the DEKs do the actual bulk AES-256 encryption

### RDS Proxy

Fully managed database proxy for [RDS and Aurora](../services/rds-proxy.md) that pools and multiplexes connections. Core exam facts:

- **Connection pooling + multiplexing** — many client-to-proxy connections onto **fewer** proxy-to-database connections. Fixes **"too many connections"** on small/burstable (t2, t3) instances and eliminates Lambda per-invocation connection overhead
- **Transparent failover** — clients hold the **proxy endpoint** and **wait** rather than error; the proxy re-points its pool to the new primary in the background, **bypassing DNS caching** to cut failover time **up to ~66%**
- **Fully managed** — auto-scaling and HA by default; you don't size, scale, or patch it
- **VPC-only** — never publicly accessible; reach it from within a VPC or a VPN/Direct Connect network
- **Engines** — RDS MySQL/MariaDB/PostgreSQL/SQL Server + Aurora MySQL/PostgreSQL. **No Oracle**, no self-managed EC2 databases
- **Secrets Manager is required** for the proxy-to-database credentials, even when clients authenticate via IAM — the pairing behind *"no hard-coded credentials"*
- **Aurora read-only endpoints** pool across replicas; plain-RDS proxies target the **primary only** (not read replicas)

> [!tip] Exam Tip
> Pattern-match: *"too many connections" / burstable instance connection exhaustion* → **RDS Proxy**. *"Lambda + RDS connection overhead"* → **RDS Proxy**. *"reduce/transparent failover without app changes"* → **RDS Proxy**. *"no hard-coded DB credentials"* → **RDS Proxy + Secrets Manager (+ IAM auth)**. *"publicly accessible proxy"* or *"Oracle + proxy"* → **NOT RDS Proxy**.

> [!warning] Gotcha
> **Multiplexing isn't guaranteed** — **session pinning** (session-state changes, prepared statements, temp tables, statements over 16 KB) disables it for the rest of the session, silently negating the pooling benefit.

## DMS

See [DMS](../services/dms.md) for full details.

- **DMS = managed database migration and replication.** Pattern-match: database migration to AWS, from AWS, or between AWS-hosted endpoints.
- **Minimal downtime = Full Load + CDC.** DMS loads the existing data, captures changes made during the load, applies cached changes, then keeps replicating until cutover.
- **DMS Standard** uses a **replication instance** in a [VPC](../services/vpc.md), source/target endpoints, and replication tasks. A replication instance can run multiple tasks but must be sized for the combined load.
- **DMS Serverless** uses serverless replications and **DCUs** instead of a provisioned replication instance. Use this wording when the question mentions automatic provisioning/scaling or no instance management.
- **Homogeneous migration** means same or compatible engine, such as MySQL to [RDS](../services/rds.md) MySQL. No cross-engine schema conversion is needed.
- **Heterogeneous migration** means engine change, such as Oracle to [Aurora](../services/aurora.md) PostgreSQL. Use **DMS Schema Conversion** or **AWS SCT** for schema/code conversion, then DMS for data movement and CDC.
- **On-premises to on-premises is not supported.** Current AWS FAQ wording requires the source or target database, or both, to reside in RDS or on EC2.
- **Large constrained-network migrations** can pair SCT/Snowball/[S3](../services/s3.md) for the baseline load with DMS CDC for the catch-up stream.

| Trigger                                       | Answer                                    |
| --------------------------------------------- | ----------------------------------------- |
| "Migrate database with minimal downtime"      | **DMS Full Load + CDC**                   |
| "Replicate only changes; data already loaded" | **CDC Only**                              |
| "One-time copy; downtime acceptable"          | **Full Load**                             |
| "Replication instance"                        | DMS Standard                              |
| "Serverless replications" / "DCUs"            | DMS Serverless                            |
| "Same database engine"                        | Homogeneous; no SCT/schema conversion     |
| "Different database engine"                   | DMS Schema Conversion or AWS SCT plus DMS |
| "Both databases are on-premises"              | Not supported by DMS                      |
| "Multi-TB migration, network too slow"        | Snowball/S3 baseline plus DMS CDC         |

> [!warning] Gotchas
>
> - Older material often says **SCT is outside DMS**. Current AWS also has **DMS Schema Conversion** in the DMS console, while the downloadable **AWS SCT** remains broader for data warehouses, big-data frameworks, application SQL, and ETL.
> - **CDC Only** does not copy historical data. It only replays changes after a baseline load done by another method.
> - **Fleet Advisor is historical** for current study: AWS ended support on **May 20, 2026**.

## Aurora

See [Aurora](../services/aurora.md) for full details. Treat Aurora as a **separate product** from [RDS](../services/rds.md) — different storage model, different endpoints, different billing.

### Core architecture facts

- **Cluster** = 1 primary + **0–15 replicas** + a **shared cluster volume**. Replicas serve **reads and failover simultaneously** (RDS makes you pick standby *or* read replica); any of the 15 can be promoted, and failover is fast because **no storage moves**
- Storage: **6 copies across 3 AZs**, replicated **synchronously at the storage level** (4-of-6 quorum, no instance CPU cost), **SSD only**, **self-healing** (disk failure rarely needs a restore)
- **Max volume 128 TiB** on the exam (current AWS: **256 TiB** since Jul 2025 on recent engine versions)
- Billing: storage is **consumed GB-month** (never allocated). **Aurora Standard** adds per-request read/write I/O charges; **Aurora I/O-Optimized** has no separate read/write I/O operation charges. Compute is per second with a **10-minute minimum**; free backup allowance = **100% of cluster storage**; **no free tier** (no micro instances)
- **Endpoints**: **cluster** (always the primary, R/W) · **reader** (load-balances replicas, auto-updates as they come and go) · custom (subset) · instance (one specific instance)

### Backtrack vs fast clone vs restore

| Feature                     | What it does                                                                                      | Key limit                                 |
| --------------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| **Backtrack**               | **In-place rewind** of the *same* cluster to a past time — endpoints unchanged                    | Enable per cluster; **Aurora MySQL only** |
| **Fast clone**              | New DB **referencing** source storage (copy-on-write) — near-instant, stores **only differences** | —                                         |
| **Restore** (snapshot/PITR) | Always a **new cluster** + new endpoints                                                          | Same as RDS                               |

### Aurora Serverless

- **ACUs** (~2 GiB memory each); set a **min/max ACU** range; per-second compute billing; **pause to zero = storage-only cost**. "Fargate for databases"
- Fits: infrequent, new, variable, or unpredictable workloads; dev/test; multi-tenant apps
- **v1** (what older questions describe): warm pool of shared stateless ACUs + AWS-managed **proxy fleet** brokering connections (scaling never drops connections). **Retired March 31, 2025**
- **v2 / Aurora serverless** (AWS renamed v2 to Aurora serverless in April 2026): a serverless **instance type inside a normal Aurora cluster**; **0.5-ACU** increments; **0–256 ACUs**; scale-to-zero/auto-pause since Nov 2024 (resume ~15 s); full Multi-AZ/Global Database/reader parity; **Data API supported** (relaunched for v2 *and* provisioned)

> [!warning] Gotcha
> **Reserved Instances never cover Aurora Serverless** — that answer is a distractor. **Database Savings Plans** (Dec 2025, up to ~35%) are the commitment discount that does cover Aurora serverless.

### Aurora Global Database

- 1 primary region (1 writer + up to 15 readers) replicating to **up to 5 (exam) / 10 (current, May 2025) secondary regions**, each with **up to 16 read-only replicas** (the extra slot is the freed writer slot, not a different limit)
- Replication is **storage-layer** (no performance impact), **~1 s**, **one-way**, and **asynchronous cross-region** — in-region across-AZ replication is the synchronous leg. That async leg is why cross-region **RPO ≈ 1 s, not zero**
- **Switchover** = planned, syncs first, **RPO 0**; **failover** = unplanned, RPO = replication lag and RTO on the order of **minutes**; promote a secondary to read/write
- Two use cases: **cross-region DR/business continuity** (low RPO + RTO) and **global low-latency reads**

> [!tip] Exam Tip
> Triggers: *"region-wide outage" / "business continuity" / "low RPO+RTO across regions" / "low-latency reads for international users"* → **Global Database**. Traps: secondaries are **read-only until promoted**; replication is **one-way**, and **async** on the cross-region leg.

### Aurora Multi-Master (retired)

Concept-level only — it existed **only on Aurora MySQL v1 (5.6), EOL Feb 28, 2023**:

- **All instances write**; **no cluster or reader endpoint** — the **application** connects directly and load-balances/reroutes itself
- Writes are **quorum-based**: storage nodes reject conflicting in-flight changes; a rejected quorum returns an **error the app must handle**
- Payoff is **fault tolerance** (operate through a writer failure with zero promotion delay), **not write scaling** — and only if the app is designed for it

## AWS Backup

See [AWS Backup](../services/aws-backup.md) for full details. A **high-level awareness** topic for SAA: know *what it does* and *when to choose it*.

- **Fully managed, centralized data protection** — consolidates backup **storage and configuration** (frequency + retention) across services, across **accounts** (via [Organizations](../services/organizations.md) / [Control Tower](../services/control-tower.md)), and across **regions** (cross-region copy)
- Components: **backup plan** (frequency / window / lifecycle / vault / region copy), **resource** (what is backed up), **vault** (destination, at least 1 required)
- **On-demand backups** exist too — you are not limited to plans; cron scheduling goes down to **hourly** at most
- **Continuous backups enable PITR** for select services only (e.g. [S3](../services/s3.md), [RDS](../services/rds.md))
- **Cold storage**: minimum **90 days**; retention must be at least 90 days beyond the transition; early deletion incurs a pro-rated charge; only **eligible resource types** ([EBS](../services/ebs.md), [EFS](../services/efs.md), [DynamoDB](../services/dynamodb.md), Timestream, VMware) — the transition is **ignored** for others
- **Vault Lock** = WORM on a vault. **Governance mode**: removable with sufficient IAM permissions. **Compliance mode** (`ChangeableForDays`, minimum 3 days / 72 hours grace): after grace time, **immutable — not even AWS can delete**. Retention still applies; backups still age out

> [!tip] Exam Tip
> Triggers: *"centrally manage backups across accounts"* means **AWS Backup + Organizations/Control Tower**; *"copy backups to another region for DR"* means **AWS Backup region copy**; *"immutable AWS Backup recovery points"* means **Backup Vault Lock**.

> [!warning] Gotcha
> **Backup Vault Lock ≠ S3 Glacier Vault Lock ≠ [S3 Object Lock](#s3-object-lock)** — three different WORM features on three different resources. And **cold storage is not universal**: don't assume every AWS-Backup-supported service can tier to cold.

## Database Transaction Models — ACID & BASE

See [Database Transaction Models](../concepts/db-transaction-model.md) for the distinction between transaction guarantees, read freshness, and partition behavior.

- **CAP:** during a network partition, a distributed system cannot guarantee both linearizable consistency and availability for every operation; “pick two” is an incomplete shortcut.
- **ACID:** atomicity, consistency, isolation, durability. ACID's valid-state consistency is different from CAP's read-ordering consistency.
- **BASE:** basically available, soft state, eventually consistent; applications tolerate temporary replica disagreement. This is not a rule that NoSQL databases have no transactions.
- **[DynamoDB](../services/dynamodb.md)** supports eventually consistent and strongly consistent table reads, plus ACID transactions. A strongly consistent read does not make a subsequent update atomic with that read; use conditions or a transaction.

> [!tip] Exam Tip
> Relational joins/constraints suggest [RDS](../services/rds.md) or [Aurora](../services/aurora.md). Key-value/document access suggests [DynamoDB](../services/dynamodb.md). ACID changes within [DynamoDB](../services/dynamodb.md) suggest Transactions; check Region scope and global-table mode.

## DynamoDB

See [DynamoDB](../services/dynamodb.md) and [NoSQL Access Patterns](../concepts/nosql-access-patterns.md) for architecture, diagrams, and worked examples.

- **Table/item/attribute:** fixed primary-key schema, flexible other attributes; **400 KB per item** including names and values. Simple PK is unique; composite PK + SK pair is unique.
- **Regional and multi-AZ by default:** use [global tables](../services/dynamodb.md#global-tables) for cross-Region replication. [Gateway endpoints](../services/vpc-gateway-endpoints.md) give private VPC workloads a route to the regional service.
- **Capacity:** ordinary write units round to **1 KB**; strongly consistent reads to **4 KB**; eventually consistent reads cost half; transactional writes cost twice ordinary writes, and transactional reads cost twice strongly consistent reads. Rate calculations use **operations per second**. Ten separate 2.5 KB items/second need **30 WCU**, **10 RCU for strongly consistent reads**, or **5 RCU for eventually consistent reads**.
- **On-demand:** pay per request unit; useful for unknown/variable load. **Provisioned:** allocate RCU/WCU, optionally with target-tracking auto scaling. Both can throttle on hot keys; auto scaling is not instantaneous.
- **Query:** equality on a partition-key value, optionally narrowed by sort key. **Scan:** evaluates across a table/index. Filters and response projections do not save capacity for evaluated items. Follow pagination even after an empty filtered page.
- **LSI:** same PK, alternate SK; create with table; strongly consistent or eventually consistent reads; shares table capacity; **5 per table**, **10 GB per item collection**.
- **GSI:** alternate PK and optional SK; add/delete later; eventually consistent reads only; independent provisioned capacity; **20 default quota**. No automatic fetch of unprojected attributes. Sparse indexes include only items with required index-key attributes.
- **Streams:** item-change records retained **24 hours**, ordered per item. Four views: keys, old image, new image, both images. [Lambda](../services/lambda.md#event-source-mapping-esm) polls through an event source mapping and can process records more than once; make side effects idempotent.
- **Global tables:** multi-active reads/writes. **MREC** is asynchronous with last-writer-wins conflict resolution; a strongly consistent local read does not guarantee freshness of remote writes. **MRSC** supports global strongly consistent reads through synchronous replication; exactly three participating Regions, with three replicas or two plus a witness. MRSC does not support TTL, LSIs, or transaction APIs; GSI reads remain eventually consistent.
- **DAX:** VPC-based cache for repeated eventually consistent reads; item and query caches are distinct. Strongly consistent reads pass through uncached. Write-through updates the item cache, not cached query results.
- **TTL:** Number in epoch **seconds**; delayed background deletion, not exact expiry enforcement. Initial delete consumes no write capacity; MREC replicated deletes are charged.
- **PITR:** opt-in **1–35 days**, restore to a new table at second granularity; latest restorable time typically trails now by about five minutes. Replication does not replace backups.

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | Managed key-value/document database | [DynamoDB](../services/dynamodb.md) |
> | Alternate sort order with strongly consistent reads | LSI, planned at table creation |
> | New index after creation or different grouping key | GSI |
> | Repeated reads need microsecond latency and tolerate staleness | DAX |
> | React to inserts, modifications, or deletions | Streams plus Lambda |
> | Multi-Region writes | Global tables; choose MREC/MRSC from consistency needs |
> | Expire old items automatically | TTL, with application-side validity checks |
> | Roll back accidental item changes | PITR restore to a new table |

> [!warning] Gotcha
> GSI “global” does not mean global tables. Strong consistency does not mean a multi-item transaction. PartiQL gives [DynamoDB](../services/dynamodb.md) SQL-compatible syntax, not a relational engine. A single small eventually consistent read may consume **0.5** read unit; the “minimum one RCU” shortcut is wrong for that case.

## ElastiCache

See [ElastiCache](../services/elasticache.md) for the caching and session-state diagrams, engine comparison, and deployment options.

- **In-memory, managed Valkey / Redis OSS / Memcached.** Sub-millisecond access; holds temporary copies of data, not the durable system of record. AWS recommends **Valkey** for new deployments (priced 20% lower node-based, 33% lower serverless).
- **Use cases:** **read-heavy** workloads, **low-latency** requirements, reducing database cost, and **session state** for stateless servers behind a load balancer and Auto Scaling group.
- **Requires application changes.** The application checks the cache, falls back to the database on a miss, populates the cache, and handles invalidation. "No code changes allowed" usually rules ElastiCache out.
- **Caching strategies:** **lazy loading** caches only requested data but can be stale and costs three trips per miss; **write-through** keeps the cache fresh but leaves new nodes missing data and causes churn; a **TTL** bounds staleness and wasted space.
- **Memcached:** simple data types, **no replication**, data split across multiple nodes, **no backups** on node-based clusters, **multi-threaded**.
- **Valkey / Redis OSS:** advanced data structures (sorted-set leaderboards, pub/sub, geospatial), **Multi-AZ** with automatic failover, **replicas to scale reads** (up to 5 per shard), **backup and restore**, **transactions**. Cluster mode enabled also shards data (up to 500 shards).
- **Global Datastore:** cross-Region replication for node-based Valkey/Redis OSS to up to two secondary Regions; secondaries are read-only and promotion is manual.
- **ElastiCache Serverless:** no capacity planning, replicates across multiple AZs, always encrypted, billed by data stored and ECPUs.

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | Read-heavy, reduce database load or cost, low latency | [ElastiCache](../services/elasticache.md) in front of the database |
> | Users lose sessions behind an ALB; make instances stateless | Sessions in ElastiCache or DynamoDB |
> | Cache needs HA, failover, backup/restore, persistence, or advanced data types | Valkey / Redis OSS |
> | Simplest cache, multi-threaded, simple values | Memcached |
> | Cache data must never be stale | Write-through, usually with TTL |
> | Cache DynamoDB reads with minimal code change | DAX, not ElastiCache |

> [!warning] Gotcha
> Valkey/Redis OSS transactions run commands in sequence without interleaving, but they **do not roll back** if one command fails at execution time; they are not relational ACID transactions. Sharding is not Memcached-only: Valkey/Redis OSS cluster mode enabled also partitions data. Memcached's real gap is **replication and backups**.

## Amazon Neptune

See [Neptune](../services/neptune.md) for the graph data model diagram, an openCypher fraud-detection example, and the comparison with other AWS databases.

- **Managed graph database.** Relationships are as important as the data. **Nodes** are things with a label and key-value properties; **relationships** are directional and named, can change freely, and can be followed in either direction.
- **Why not relational:** relationships in a relational database are designed in advance, and each hop is another join, which is expensive and hard at scale.
- **Architecture like [Aurora](../services/aurora.md):** a primary plus up to **15 read replicas** that also act as failover targets, on a shared cluster volume with six copies across three AZs. Multi-AZ means replicas in other AZs; read scaling means adding replicas.
- **Private in a VPC by default.** Public endpoints are opt-in per instance and require **IAM database authentication**.
- **Continuous backups** to S3 with **point-in-time recovery** (1–35 days retention).
- **Neptune Serverless** scales in **NCUs** (about 2 GiB of memory each, 1–128). **Global Database** has one writer Region and up to **five read-only secondary Regions**. **Neptune Analytics** is a separate in-memory engine for graph algorithms and vector search, not the transactional database.
- **Query languages:** Gremlin and openCypher (property graph), SPARQL (RDF).

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | "Graph", nodes and relationships, fluid or ever-changing relationships | [Neptune](../services/neptune.md) |
> | Social network, followers, friends of friends | Neptune |
> | Fraud detection by linking people, addresses, and cards | Neptune |
> | Recommendations based on connections between users and products | Neptune |
> | Network or IT dependency mapping, drug interactions, knowledge graphs | Neptune |
> | Graph algorithms or GraphRAG context for generative AI | Neptune Analytics |
> | Key-value lookups at massive scale | DynamoDB, not Neptune |

## Amazon OpenSearch Service

See [OpenSearch](../services/opensearch.md) for the architecture diagram, ingestion paths, and the comparison with Athena and CloudWatch Logs Insights.

- **Managed search and analytics.** Runs OpenSearch and OpenSearch Dashboards, forked from **Elasticsearch** and **Kibana**; the service was called Amazon Elasticsearch Service until 2021, so older questions use those names.
- **Two jobs, both near real time:** **full-text search** and **log analytics**. JSON documents are indexed on arrival, every field is searchable, and a domain makes new data searchable within about a second.
- **Domain:** you choose instance types and node counts, pay per instance-hour whether or not it is queried, and there is no automatic scaling of node count. In a VPC or public (chosen at creation). Multi-AZ across 2 or 3 AZs, which protects data only if indexes have **replica shards**; **Multi-AZ with Standby** uses 3 AZs with one on standby and fails over in under a minute. **UltraWarm** and **cold storage** keep older indexes searchable at lower cost.
- **Serverless:** no nodes; set minimum and maximum **OCUs** (6 GiB each). Since the next generation became GA in May 2026, a collection group can **scale to zero** after 10 idle minutes, with a 10 to 30 second cold start; a non-zero minimum avoids it. Refresh is about 10 seconds.
- **Sources:** **Firehose** and **CloudWatch Logs** subscription filters deliver directly (the CloudWatch Logs path uses a Lambda function behind the scenes); Kinesis Data Streams, S3, and DynamoDB need **Lambda** in between, or use **OpenSearch Ingestion**, a managed pipeline that filters and transforms with no servers.
- **Outputs:** **OpenSearch Dashboards** (built-in UI: search, visualize, alert), **Quick Sight**, or custom applications. They all query the index; none of them receives the data.

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | Search or log analytics in **near real time** | [OpenSearch](../services/opensearch.md) |
> | Elasticsearch, Kibana, or ELK as a managed service | OpenSearch |
> | Real-time clickstream analysis and dashboards | Kinesis, then OpenSearch, then Quick Sight |
> | Search VPC Flow Logs or CloudTrail with dashboards and alerts | CloudWatch Logs or Firehose into OpenSearch |
> | Ingest and transform from many sources with no Lambda or servers | OpenSearch Ingestion |
> | Semantic search or a vector store for RAG | OpenSearch vector search |
> | Store cheaply and query occasionally | [S3](../services/s3.md) plus [Athena](../services/athena.md), not OpenSearch |

## Scaling — Horizontal vs Vertical

See [Scaling — Horizontal vs Vertical](../patterns/scaling.md) for the full treatment.

- **Vertical scaling = resize one instance.** Simple, works for monoliths, no app changes. But requires **stop / modify / start** (disruption), only happens in maintenance windows, and is **capped at the largest available instance size**.
- **Horizontal scaling = add more instances behind a load balancer.** No upper limit, no disruption, finer granularity, and usually cheaper per unit of capacity. Requires a **load balancer** and **stateless application design**.
- **Sessions are the critical constraint.** With horizontal scaling, a customer's consecutive requests can hit different instances. If session state lives on the instance, the user appears logged out, the cart empties, the wizard restarts. **Move sessions off-host** to [ElastiCache](../services/elasticache.md#session-state-for-stateless-servers) (Valkey, Redis OSS, or Memcached), [DynamoDB](../services/dynamodb.md), or RDS.
- **ALB sticky sessions are NOT the proper fix.** They pin a user to one instance; if that instance is replaced or scaled in, the session is lost anyway. Sticky sessions are a migration aid for legacy apps. Exam-correct long-term answer: **off-host sessions, stateless instances**.
- **Resizing EC2 is a stop/start in disguise** — instance-store data destroyed, auto-public-IPv4 changes. See [EC2 — Stop/Start vs Restart](../services/ec2.md#stopstart-vs-restart--these-are-not-the-same).
- **Horizontal scaling is the foundation for HA.** Vertical scaling cannot reach fault tolerance — a single instance always inherits the resilience tier of its AZ. Horizontal + ELB + Auto Scaling Groups across multiple AZs is the HA pattern.

### Scaling — pattern-match cheat-sheet

| Question pattern                                                | Answer                                                                                                                             |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| "App is a monolith on one instance, traffic doubled"            | **Vertical scaling** — resize to a larger instance type                                                                            |
| "Cannot tolerate downtime, need to scale during business hours" | **Horizontal scaling** — adding instances doesn't disrupt existing connections                                                     |
| "Users get logged out at random after deploying behind ALB"     | **Move sessions off-host** to [ElastiCache](../services/elasticache.md) / [DynamoDB](../services/dynamodb.md), not sticky sessions |
| "Shopping cart empties intermittently across instances"         | **Stateless app + external session store**                                                                                         |
| "Legacy app, on-host sessions, no time to refactor"             | **ALB sticky sessions** — migration aid only                                                                                       |
| "Need infinite scale"                                           | **Horizontal** — vertical has a hard cap                                                                                           |
| "Need fault tolerance, not just scalability"                    | **Horizontal scaling across multiple AZs** — load balancer + Auto Scaling Group; vertical can't reach this                         |
| "Cheapest at large total capacity"                              | **Many small commodity instances (horizontal)** — large instances carry a price premium                                            |

> [!tip] Mnemonic
> **Many Bobs = horizontal. One giant Bob = vertical.** Adding workers = scale out; growing one worker = scale up.

## ELB — Elastic Load Balancer Architecture

See [ELB](../services/elb.md) for the full treatment.

- **Family**: CLB (v1, legacy — never the answer for new deployments), ALB + NLB (v2, support target groups + rules). The real exam skill is **ALB vs NLB**: HTTP/HTTPS/WebSocket means **ALB** (true L7); TCP/TLS/UDP (email, SSH, gaming, custom protocols) means **NLB** (L4).
- **A load balancer is nodes, not a box.** One or more nodes deploy into one subnet per chosen AZ; a single **A record** resolves to all active nodes; DNS responses tend to distribute clients across nodes; nodes auto-scale and are HA.
- **Internet-facing vs internal changes node IP addressing only**: internet-facing nodes get public + private IPs; internal nodes get private IPs only. Everything else is identical.
- **Internet-facing LB can target both public and private instances** — backend targets never need public IPs; the nodes only need to reach the targets.
- **Subnet rules**: one subnet per AZ; ALB requires 2+ AZs; internet-facing requires public subnets; **8+ free IPs per subnet, /27 recommended** (/28 is the strict minimum).
- **NLB specials**: static IPs via Elastic IP per AZ (ALB/CLB have none — DNS name only); can technically run in a single AZ, but default to "2+ AZs" answers.
- **Cross-zone load balancing**: lets every node distribute to all instances in all AZs. **ALB: on by default at the LB level; target groups can override.** Classic: historically off.
- **ALB is L7 HTTP/HTTPS only** — cannot listen on TCP/UDP/TLS or handle non-web protocols. Rules (priority-ordered, default catch-all last) match on host / headers / method / path / query / source IP and can forward, redirect, return a fixed response, or **authenticate (OIDC / Cognito)**. Health checks are application-aware.
- **ALB always terminates SSL/TLS** — a cert must live on the ALB; the backend connection is separate (hop-by-hop at best, never unbroken).
- **NLB is L4 (TCP/TLS/UDP)** — no header/cookie visibility, no HTTP stickiness, no path/host routing; millions of requests per second at roughly a quarter of ALB latency.
- **NLB TCP listener = pass-through** — the canonical unbroken end-to-end encryption answer. The NLB **TLS listener terminates** (needs a cert) — don't conflate them.
- **PrivateLink = NLB.** High-value exam fact.
- **Decision framework — memorise the 5 NLB reasons, else default to ALB**: (1) unbroken end-to-end encryption, (2) static IPs for whitelisting, (3) absolute best performance, (4) non-HTTP/HTTPS protocols, (5) PrivateLink.

### ELB — pattern-match cheat-sheet

| Question pattern                                                                      | Answer                                                                                                     |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| "Load balance HTTP/HTTPS/WebSocket"                                                   | **ALB** (Layer 7)                                                                                          |
| "Load balance email / SSH / gaming / custom TCP or UDP protocol"                      | **NLB** (Layer 4)                                                                                          |
| "Route by path / host / headers / cookies; HTTP redirect; built-in OIDC/Cognito auth" | **ALB** — rules are L7-only                                                                                |
| "Unbroken / end-to-end encryption from client to instances"                           | **NLB with a TCP listener** — never ALB (always terminates), and not the NLB TLS listener (terminates too) |
| "Millions of requests per second / lowest latency / highest performance"              | **NLB**                                                                                                    |
| "Expose a service to other VPCs via PrivateLink"                                      | **NLB**                                                                                                    |
| "Need static IP / Elastic IP / whitelist a fixed IP for the LB"                       | **NLB** — the only type with static IPs                                                                    |
| "Performance-critical AND path-based routing in one question"                         | **ALB** — L7 routing forces ALB; you can't have both on one LB                                             |
| "Application-aware health checks (verify the app responds)"                           | **ALB** — NLB checks are shallow (ICMP / TCP handshake per the course)                                     |
| "Uneven load across AZs (one instance overloaded)"                                    | **Enable cross-zone load balancing**                                                                       |
| "Minimum subnet size for a load balancer"                                             | **/27** if offered (AWS-recommended); **/28** only if /27 absent (satisfies the 8-free-IP rule)            |
| "Do backend instances behind an internet-facing LB need public IPs?"                  | **No** — targets can stay private                                                                          |
| "Tiers must scale independently without reconfiguration"                              | Insert an **internal load balancer** between tiers                                                         |
| "Migrating off a Classic Load Balancer" / "one SSL cert per LB / no SNI"              | Move to **ALB** (SNI = many certs on one LB; or NLB per protocol)                                          |

> [!warning] Gotcha — node distribution vs instance distribution
> DNS responses tend to distribute clients across **nodes**; over many clients, that approximates an even split. Cross-zone only changes what a node does with its share — whether it can send to instances in other AZs.

### ELB — SSL connection modes and session stickiness

See [SSL/TLS connection modes](../services/elb.md#ssltls-connection-modes--bridging-pass-through-offload) and [session stickiness](../services/elb.md#session-stickiness) on the ELB page.

- **Three connection modes** — the recurring theme is *who holds the certificate, who does the crypto, what the LB can see*:
  - **Bridging** (ALB default): HTTPS terminated on the ELB, **re-encrypted** to instances. Certs needed on the ELB **and every instance** (the second half is easy to forget). Termination is what makes L7 routing possible.
  - **Pass-through** (NLB, TCP listener): never decrypted, one tunnel client-to-instance. Cert on **instances only** — the only mode where **AWS never touches your private key**. Pairs with **CloudHSM**.
  - **Offload**: HTTPS terminated on the ELB, **plain HTTP** behind. No certs or crypto on instances (smaller instance types), but data crosses the AWS network **in plaintext**.
- **The listener protocol determines the mode, not the LB type** — NLB has supported TLS termination since January 2019. TCP listener = pass-through; HTTPS/TLS listener = terminated.
- **Stickiness is a target-group setting** — not a listener setting, not a load-balancer setting. LB-generated **`AWSALB`** cookie, duration **1 second to 7 days**; breaks on **instance failure** or **cookie expiry**; causes **uneven backend load**.
- **Two stickiness types**: duration-based (`AWSALB`) and application-based (**`AWSALBAPP-`** cookie, app-set, February 2021). **CLB** uses **`AWSELB`**.
- **Lost carts / repeated logins has two answers**: stickiness is the *quick fix*; externalising state to [DynamoDB](../services/dynamodb.md) / ElastiCache is the *well-architected* answer — "best practice" / "scalable" / "even load distribution" wording points to stateless.

| Trigger phrase                                                                                   | Answer                                                                 |
| ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| "Certificate must not be exposed to AWS" / "regulatory control of private keys" / "CloudHSM"     | **Pass-through** (NLB, TCP listener)                                   |
| "Route based on URL path / host header / HTTP headers" + HTTPS                                   | **Bridging** (ALB, HTTPS listener)                                     |
| "Reduce cryptographic overhead on instances" / "smaller instance types" / "no certs on backends" | **Offload**                                                            |
| "Default mode of the Application Load Balancer"                                                  | **Bridging**                                                           |
| "End-to-end encryption"                                                                          | **Pass-through** (or bridging — read whether the LB needs to see HTTP) |
| "Listener configured for TCP"                                                                    | **Pass-through** — no decryption                                       |
| "Listener configured for HTTPS"                                                                  | Terminated on the LB — bridging *or* offload                           |
| Compliance requires encryption in transit "end-to-end" / "at all times"                          | **Offload is wrong** (plaintext on the AWS network)                    |
| "Users being logged out" / "shopping cart emptied" / "session lost"                              | **Stickiness** (quick fix) or **externalise state** (best practice)    |
| "AWSALB cookie"                                                                                  | ALB **duration-based** stickiness                                      |
| "AWSELB cookie"                                                                                  | **CLB** stickiness                                                     |
| "AWSALBAPP-" cookie                                                                              | ALB **application-based** stickiness                                   |
| "Uneven load across backend instances" (stickiness in scenario)                                  | Stickiness is the **cause**                                            |
| "Stateless" / "session state in [DynamoDB](../services/dynamodb.md)"                             | No stickiness needed — fair balancing                                  |
| "1 second to 7 days"                                                                             | Stickiness duration range                                              |

## EC2 Auto Scaling — Auto Scaling Groups (ASG)

See [EC2 Auto Scaling](../services/ec2-auto-scaling.md) for the full treatment.

- **Division of labour**: the ASG controls the **when** and **where** (launch timing, subnets/AZs); the [launch template](../services/ec2.md#launch-configurations--launch-templates) controls the **what** (AMI, type, bootstrap). Hooks and policies attach to the **ASG**, not the template.
- **Min / Desired / Max** = floor / target / ceiling. The ASG's one job: keep running count equal to desired. **Min/max always clamp** whatever a policy calculates.
- **Scaling policies are optional** — a policy-free ASG just holds static values. Manual scaling = editing min/desired/max yourself (also a cost-control pin).
- **Scheduled scaling** for predictable patterns ("every day at 9am", "sales event").
- **Dynamic scaling hierarchy — target tracking first, then step, then simple**: target tracking holds a metric at an ideal value ("thermostat": 4 predefined metrics — `ASGAverageCPUUtilization`, `ASGAverageNetworkIn`/`Out`, `ALBRequestCountPerTarget` which is **ALB only** and needs `ResourceLabel`); step scaling reacts in graduated steps by breach size; simple scaling is legacy — fixed adjustment and **cooldown blocks further response** while an activity runs (step keeps responding — why AWS recommends step over simple even for a single adjustment).
- **Who owns the alarm?** You create/manage alarms for simple and step; **AWS creates and manages target tracking's alarms** — never touch them.
- **Cooldown vs instance warmup**: cooldown (default 300 s) is **simple-scaling-specific**; target tracking and step use **default instance warmup** (delay before a new instance counts toward metrics), falling back to the cooldown value if unset.
- **[SQS](../services/sqs.md) worker pool**: scale on `ApproximateNumberOfMessagesVisible`.
- **Self-healing**: failed (or manually terminated) instances are replaced to restore desired capacity. **Cheap HA trick**: min = desired = max = 1 across multiple AZs relaunches a lone instance, potentially in another AZ — unlike [EC2 auto recovery](../services/ec2.md#status-checks-and-auto-recovery), which is same-AZ only.
- **ELB integration**: instances are auto-registered/deregistered with the target group; the ASG can use **application-aware ELB health checks** instead of infrastructure-only EC2 status checks.
- **Lifecycle hooks**: pause the launch or terminate transition (`Pending:Wait`, `Terminating:Wait`) for custom actions. Only 2 transitions: `EC2_INSTANCE_LAUNCHING` and `EC2_INSTANCE_TERMINATING`. Default timeout **3600 s** (`HeartbeatTimeout` range 30–7200 s; global cap 48 h or 100 times the heartbeat, whichever is smaller). Resume early with `CompleteLifecycleAction`; extend with `RecordLifecycleActionHeartbeat`. `DefaultResult` API default is **`ABANDON`**, and **`ABANDON` never rescues an instance** — on launch it terminates and replaces it; on terminate it just skips remaining hooks. Notifications go to **SNS or SQS**; **EventBridge** for event-driven automation.
- **Standby** exempts 1 instance from all ASG activity (maintenance); individual scaling **processes** (Launch, Terminate, AZRebalance, HealthCheck, ReplaceUnhealthy, …) can be suspended and resumed.
- **Cost**: the ASG itself is free; prefer **more, smaller instances** for granular scaling steps.

### ASG — pattern-match cheat-sheet

| Question pattern                                                                             | Answer                                                            |
| -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| "Maintain average CPU at 40%" / "keep metric at a value" / "set and forget" / "thermostat"   | **Target tracking**                                               |
| "Respond differently by how large the breach is" / "graduated response" / "granular control" | **Step scaling**                                                  |
| "Policy stopped responding while a scaling activity was in progress"                         | **Simple scaling** cooldown is the culprit; recommend **step**    |
| "Known/predictable traffic pattern" / "every day at 9am"                                     | **Scheduled scaling**                                             |
| "Worker pool" / "queue depth" / "process backlog"                                            | **SQS-based scaling** on `ApproximateNumberOfMessagesVisible`     |
| "Requests per instance behind an ALB"                                                        | **Target tracking** with `ALBRequestCountPerTarget` (ALB only)    |
| "AWS manages the alarm for you"                                                              | **Target tracking** (you own simple/step alarms)                  |
| "Scaling keeps flapping/thrashing"                                                           | Widen the **dead band** between scale-out and scale-in thresholds |
| "Warm-up delay before a new instance counts toward metrics" (target tracking/step)           | **Default instance warmup**, not cooldown                         |
| "Cheapest HA for a single EC2 instance"                                                      | ASG with **min = desired = max = 1** across multiple AZs          |
| "Run a script before the instance receives traffic" / "pre-load or warm cache"               | **Launch lifecycle hook** (`Pending:Wait`)                        |
| "Copy logs off / back up / drain / graceful shutdown before termination"                     | **Terminate lifecycle hook** (`Terminating:Wait`)                 |
| "Need more time in the hook wait state"                                                      | `RecordLifecycleActionHeartbeat`                                  |
| "Signal the hook is done / resume immediately"                                               | `CompleteLifecycleAction`                                         |
| "Maintenance on one ASG instance without termination or scaling interference"                | Set the instance to **Standby**                                   |
| "Notify a topic when scaling occurs" / "event-driven response to scaling"                    | Lifecycle hooks with **SNS** / **EventBridge**                    |
| "Pre-initialised instances kept stopped and ready"                                           | **Warm pools** (not plain lifecycle hooks)                        |

> [!warning] Gotcha — ASG traps
>
> - **`ABANDON` on a terminate hook does not save the instance** — both results terminate it; `ABANDON` only skips remaining hooks.
> - Hook timeout numbers: default **3600 s**, single-window max **7200 s**, global max **48 h or 100 times the heartbeat, whichever is smaller** — three different numbers, don't conflate.
> - A health check that depends on a downstream database means a **DB outage churns the whole fleet** — the ASG replaces healthy instances while the real problem goes unaddressed.
> - AZ balance is **best-effort**, not guaranteed.
> - Lifecycle hooks are not user data: user data runs at boot with no coordination; a hook **holds the ASG state machine** until signalled.

## AWS Lambda

See [Lambda](../services/lambda.md) for full details.

- **Trigger keywords** for a Lambda answer: *serverless*, *event-driven*, *FaaS*, *only pay for what you use*, *short-running focused compute*
- **"Docker"** in a question means **NOT Lambda** — traditional containers point to [ECS](../services/ecs.md); Lambda *container images* (built with Docker but run inside Lambda) are a separate thing
- **>15 minutes** of work is not a single Lambda invocation (hard **900 s** timeout) — orchestrate with **Step Functions**
- **1769 MB = 1 vCPU** (vCPU is allocated indirectly via memory, 128–10240 MB); `/tmp` is 512 MB default up to 10240 MB, ephemeral
- **VPC Lambda = normal VPC resource** — needs a [gateway endpoint / NAT + IGW](../services/vpc.md#nat--nat-gateway) for outside access, and the **execution role needs EC2 network permissions** so Lambda can create ENIs
- **No CloudWatch Logs output** means a **missing execution-role permission** (`AWSLambdaBasicExecutionRole`)
- **Two-part security model** ([resource vs identity policy](../concepts/resource-vs-identity-policies.md)): **resource policy** = who can invoke (like an S3 bucket policy); **execution role** = what the function can do (like an [EC2 instance role](../services/ec2.md#instance-roles--instance-profiles))
- **Three invocation types:** **sync** (client handles retries) · **async** (Lambda retries 0–2 times, **requires idempotency**, DLQ + destinations) · **event source mapping** (polls [Kinesis](../services/kinesis.md) / [DynamoDB](../services/dynamodb.md) Streams / [SQS](../services/sqs.md) / MSK)
- **ESM permissions trap:** the execution role needs **source-service permissions** even though the code never directly reads the source — the ESM reads on the function's behalf
- **Versions immutable** (own ARN); `$LATEST` and **aliases mutable**
- **Cold start** (~100 ms, full context build) vs **warm start** (~1–2 ms, reused context); pin capacity with **provisioned concurrency**; declare reusable objects (DB connections) **outside the handler**

| If the question says...                                                                             | Think...                                                          |
| --------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| serverless / event-driven / short-running compute                                                   | **Lambda**                                                        |
| "Docker" (bare term)                                                                                | **NOT Lambda** (ECS)                                              |
| work exceeds 15 minutes                                                                             | **Step Functions** (not one Lambda)                               |
| async invoke + retries re-run                                                                       | **idempotent function**                                           |
| Lambda reads from [Kinesis](../services/kinesis.md)/SQS/DynamoDB Streams but code doesn't call them | **execution role still needs source permissions (ESM)**           |
| Lambda can't reach Aurora/RDS/EFS                                                                   | **put Lambda in the VPC** (+ EC2 network permissions on the role) |
| too many DB connections from Lambda                                                                 | **[RDS Proxy](../services/rds-proxy.md)**                         |

## EventBridge (CloudWatch Events)

See [EventBridge](../services/eventbridge.md) for full details.

- **Mental model:** *"if X happens, or at time Y, then do Z."* X = a producer service emitting an event; Y = a schedule; Z = a target (commonly [Lambda](../services/lambda.md))
- **EventBridge = CloudWatch Events "v2"** — same API, superset of features. Treat them as the **same service, different name**; EventBridge is the current/preferred name and gets all new features. It is the **evolution** of CloudWatch Events, **not** a deprecation of the API
- **Third-party / SaaS partner events** points to **EventBridge, never CloudWatch Events** — the flagship discriminator
- **Multiple / custom event buses** points to **EventBridge**; CloudWatch Events has exactly **one** default bus. Custom app events can use the default bus/API; custom buses are the EventBridge discriminator
- **Rule types:** **event pattern rule** (matches event *content*) vs **schedule rule** (matches a time)
- **Schedule expressions:** *"every N minutes"* = **rate**; *"at 8am first Monday"* = **cron**. Scheduled rules are now **"(legacy)"**, run only on the **default event bus**, and **EventBridge Scheduler** is the newer standalone scheduler that **needs no event bus**
- **Events are JSON**, delivered to the target, which can act on the contents
- **Players:** *event bus* = the stream · *rule* = the matcher · *target* = the receiver. EventBridge **observes** the bus; it is not the bus
- **CloudTrail as a source:** an API call reaches EventBridge tagged **`AWS API Call via CloudTrail`**. If the question needs the **IAM caller identity**, it must come via **CloudTrail**, not the native service event (the native event omits the principal)
- **CloudTrail events reaching EventBridge:** the CloudTrail path requires an **active trail with logging enabled**. Management events are the trail default; data and network activity events need explicit selectors. **Write/mutating** management events match an ordinary `ENABLED` EventBridge rule; **read-only** (`Describe`/`Get`/`List`) management events match only a rule set to `ENABLED_WITH_ALL_CLOUDTRAIL_MANAGEMENT_EVENTS`

| If the question says...                        | Think...                                                                                      |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------- |
| react when an EC2/resource changes state       | **EventBridge event pattern rule**                                                            |
| run a Lambda nightly / on a cron               | **EventBridge schedule rule (or Scheduler)**                                                  |
| SaaS partner events                            | **EventBridge** (not CloudWatch Events)                                                       |
| multiple / custom event buses                  | **EventBridge**                                                                               |
| react fast to an API call                      | **EventBridge on [CloudTrail](../services/cloudtrail.md) events from an active trail**        |
| react based on *who* made the API call         | **CloudTrail-path event** (`AWS API Call via CloudTrail`), not the native event               |
| react to S3 object-level / Lambda-invoke calls | **configure trail data-event selectors first**, then EventBridge                              |
| metric crosses a threshold                     | **[CloudWatch Alarm](../services/cloudwatch.md#metric-filters-and-alarms)** (not EventBridge) |

## SNS — Simple Notification Service

See [SNS](../services/sns.md) for full details.

- **Pub/sub, topic-centric:** publishers write into a **topic**, subscribers receive from it; the topic holds permissions and config. By **default every subscriber gets every message**
- **Regional service outside the VPC** — reachable through public endpoints or a private [interface endpoint](../services/vpc-interface-endpoints.md), with appropriate [IAM](../services/iam.md)/topic permissions; SNS itself is not placed in a subnet
- **Subscriber types:** HTTP/S, email, **SQS**, **[Lambda](../services/lambda.md)**, SMS, mobile push
- **Filter** = a **per-subscription** filter policy letting a subscriber opt into a **subset** — not publisher-side routing
- **Fanout** = one topic to **many SQS queues**; the answer whenever one event must reach several parallel/decoupled workloads
- **256 KB** message ceiling — notifications, not bulk data; larger payloads (up to **2 GB**) go via the **S3-backed Extended Client Library**
- **Cross-account** access to a topic = a **topic policy** (resource policy), same idea as an S3 bucket policy
- **Delivery status** is confirmable for **HTTP/S, Lambda, SQS**; failed deliveries retry per a delivery policy and are **lost unless a DLQ** (an SQS queue) is attached
- **Standard vs FIFO topic:** FIFO gives **strict ordering + dedup** but delivers to **SQS queues only** — strict ordering *plus* email/SMS/HTTP is a mismatch. FIFO throughput is **3,000 msg/s or 20 MB/s per topic**, **300 msg/s per message group**

| If the question says...                                                           | Think...                                     |
| --------------------------------------------------------------------------------- | -------------------------------------------- |
| deliver one message to multiple queues / decouple one event into parallel workers | **SNS fanout to SQS**                        |
| let a subscriber receive only relevant messages                                   | **subscription filter policy**               |
| send a message larger than 256 KB                                                 | **S3-backed Extended Client Library (2 GB)** |
| grant another account access to a topic                                           | **topic (resource) policy**                  |
| strict ordering **and** email/SMS delivery                                        | **mismatch** — FIFO topics are SQS-only      |
| which endpoints can confirm delivery                                              | **HTTP/S, Lambda, SQS**                      |
| notify a human/topic when an alarm, stack, or scaling event fires                 | **publish to an SNS topic**                  |

## SQS — Simple Queue Service

See [SQS](../services/sqs.md) for full details.

- **Public regional service**, fully managed, highly available within the region — like [SNS](../services/sns.md), it is **not** placed in a VPC subnet
- **The core loop:** send · poll · **visibility timeout** · delete. A received message is **hidden, not deleted**; if the consumer never deletes it, it **reappears** when the timeout expires. Default **30 s**, range **0 s to 12 h**. `ChangeMessageVisibility` extends it per message; setting it to **0** makes the message immediately available again
- **Delay queue vs visibility timeout** is the classic trap. **Delay queue** (`DelaySeconds`) hides a message **the moment it is added**, range **0 to 15 minutes**; the visibility timeout hides it **only after a consumer receives it**. **Message timers** are the per-message version of a delay and are **not supported on FIFO queues**. Delay longer than 15 minutes means **EventBridge Scheduler**
- **Standard vs FIFO:** Standard = best-effort ordering, **at-least-once**, nearly unlimited throughput. FIFO = strict ordering, **exactly-once into the queue**, name must end **`.fifo`**, **300 TPS unbatched / 3,000 batched** by default (high throughput mode reaches up to 70,000 TPS in the top-tier Regions)
- **"Exactly-once" is about the queue, not your code.** A consumer that crashes after doing the work but before `DeleteMessage` gets the message again — **on FIFO exactly as on standard**. Consumers must be **idempotent** either way
- **`MessageDeduplicationId`** (5-minute window, or content-based dedup) prevents producer-retry duplicates on FIFO; **`MessageGroupId`** is the ordering unit — parallelism comes from **more distinct group IDs**
- **Dead-letter queue:** when a message's **`ReceiveCount` exceeds `maxReceiveCount`** (default **10**) and it hasn't been deleted, it moves to the DLQ. A DLQ is an ordinary queue of the **same type**, in the same account and Region; **one DLQ can serve many source queues**. The **enqueue timestamp is not reset** on the move, so **set DLQ retention longer than the source's**
- **Redrive policy** (on the **source** queue: names the DLQ + `maxReceiveCount`) vs **redrive allow policy** (on the **DLQ**: `allowAll` / `byQueue` up to 10 / `denyAll`). **DLQ redrive** moves messages back to the source once the bug is fixed
- **Long polling** (`WaitTimeSeconds` up to **20 s**) is the recommended default — short polling returns immediately and burns **billable empty receives**
- **Billing is per request, not per message.** A request carries 1 to 10 messages, and **each 64 KB chunk is billed as one request**. A request returning zero messages is still billable
- **Retention** 1 minute to 14 days, **default 4 days**. **In-flight limit 120,000** for both queue types
- **Message size:** **256 KB** in course/exam material; AWS raised the real limit to **1 MiB in August 2025** ([SNS did not change](../services/sns.md)). Beyond that, **S3 plus a pointer** via the Extended Client Library, up to 2 GB
- **Fanout** = one [SNS](../services/sns.md) topic to **many SQS queues**, because an [S3 event](../services/s3.md#event-notifications) is one event to one destination. Each queue then drives its **own** ASG scaling on its **own** queue length
- **Access control:** identity policy **or** queue policy for same-account; **queue policy only** for cross-account (it is a resource policy, like an S3 bucket policy)
- **Encryption:** at rest via **KMS or SSE-SQS**; **in transit by default**

| If the question says...                                            | Think...                                                      |
| ------------------------------------------------------------------ | ------------------------------------------------------------- |
| one upload/event must start several independent parallel pipelines | **SNS fanout to multiple SQS queues**                         |
| message reappears and another worker picks it up                   | **visibility timeout** (not deleted, so it comes back)        |
| hidden the moment it is added, before any consumer sees it         | **delay queue** (`DelaySeconds`)                              |
| need a delay longer than 15 minutes                                | **not SQS** — EventBridge Scheduler                           |
| per-message delay on a FIFO queue                                  | **message timers are not supported on FIFO**                  |
| ordering matters / no duplicates at ingest                         | **FIFO queue** + `MessageDeduplicationId`                     |
| maximum throughput, duplicates tolerable                           | **Standard queue** + idempotent consumers                     |
| consumer might process the same message twice                      | **idempotency** — FIFO does not fix this                      |
| `MessageGroupId` on a **standard** queue                           | **fair queues** (tenant fairness), *not* ordering             |
| poison message / infinite reprocessing                             | **DLQ** + `maxReceiveCount`                                   |
| alert me when messages fail permanently                            | **CloudWatch alarm on the DLQ**                               |
| messages vanished from the DLQ early                               | **enqueue timestamp is not reset** — set DLQ retention longer |
| which source queues may use this DLQ                               | **redrive allow policy**, not the redrive policy              |
| reprocess after fixing the consumer                                | **DLQ redrive to source**                                     |
| reduce cost / eliminate empty receives                             | **long polling**, up to 20 s                                  |
| scale a worker fleet on backlog                                    | **ASG on `ApproximateNumberOfMessagesVisible`**               |
| payload too large for a message                                    | **S3 + pointer** (Extended Client Library, 2 GB)              |
| grant another account access to a queue                            | **queue policy** (resource policy)                            |

## Step Functions

See [Step Functions](../services/step-functions.md) for full details.

- **What it is:** serverless **orchestration**. A **state machine** is a workflow; a **state** is something that happens inside it. States take data in, modify it, and pass it out
- **Why it exists:** it works around two [Lambda](../services/lambda.md) design decisions — the **15-minute (900 s) timeout** and **stateless execution environments**. Chaining Lambdas to get past 15 minutes is the wrong answer
- **Two workflow types, fixed at creation:** **Standard** (default, **1 year** max, exactly-once, billed per state transition) vs **Express** (**5 minutes** max, at-least-once async, billed per execution plus duration and memory). The type **cannot be changed later** — you recreate the state machine
- **State types (8 in current ASL):** Task, Choice, Parallel, Map, Pass, Wait, Succeed, Fail. **Task** is the only one that performs work; the rest are flow control. (The course lists seven, omitting **Pass**)
- **Wait** pauses for a duration *or* until a date/time · **Choice** branches on input · **Parallel** runs concurrent branches · **Map** repeats actions once per item in a list · **Succeed**/**Fail** are terminal
- **Step Functions coordinates, it does not compute.** A **Task** state integrates with Lambda, AWS Batch, [DynamoDB](../services/dynamodb.md), [ECS](../services/ecs.md), [SNS](../services/sns.md), SQS, [Glue](../services/glue.md), SageMaker, EMR, and more — the *integrated service* does the work
- **Definition language is Amazon States Language (ASL)** — **JSON**-based, not YAML
- **Permissions:** the state machine **assumes an [IAM role](../services/iam.md#iam-roles)** at runtime for temporary credentials — same pattern as a Lambda execution role or an EC2 instance role
- **Triggers:** [API Gateway](../services/api-gateway.md), IoT rules, [EventBridge](../services/eventbridge.md), Lambda, or manual start

| If the question says...                                    | Think...                                    |
| ---------------------------------------------------------- | ------------------------------------------- |
| work exceeds 15 minutes / must hold state across steps     | **Step Functions**, not chained Lambdas     |
| long-running, durable, auditable, "up to one year"         | **Standard** workflow                       |
| high-volume, IoT, streaming, under 5 minutes               | **Express** workflow                        |
| "convert an existing Standard state machine to Express"    | **Not possible** — recreate it (distractor) |
| wait for a period of time or until a specific time         | **Wait** state                              |
| branch on the input                                        | **Choice** state                            |
| do the same thing for every item in a list                 | **Map** state                               |
| run several things at once                                 | **Parallel** state                          |
| actually invoke Lambda / SNS / SQS / ECS from the workflow | **Task** state                              |
| workflow definition format                                 | **ASL, JSON-based**                         |
| how the workflow gets permissions                          | **IAM role assumed at runtime**             |

## API Gateway

See [API Gateway](../services/api-gateway.md) for full details.

- **What it is:** a managed **intermediary between clients and integrations** — create, publish, and manage APIs. Highly available and scalable, and a **public service**, so it can front backends in AWS *or* on-premises
- **Three phases:** **Request** (authorize, validate, transform), **Integration** (the backend works), **Response** (transform, prepare, return)
- **API types:** HTTP, REST, WebSocket. Several features below are **REST-only** — caching and canary deployments in particular
- **Direct AWS service integration:** it can write to [DynamoDB](../services/dynamodb.md), start a [Step Functions](../services/step-functions.md) execution, or publish to [SNS](../services/sns.md) **with no backing compute**. "No Lambda needed" is the signal
- **Migration façade:** a consistent front end while the backend moves on-premises into AWS, or monolith into serverless
- **Auth:** open (no auth) · **[Cognito user pools](../services/cognito.md)** (`COGNITO_USER_POOLS` for REST; JWT authorizer for HTTP APIs) · **Lambda authorizer** (returns an **IAM policy plus a principal identifier**; API Gateway then evaluates it and either invokes the integration or returns 403) · **[IAM](../services/iam.md)** (SigV4-signed requests). The **authorizer Lambda and the integration Lambda are two different functions**
- **Endpoint types:** **Edge-optimized** (nearest CloudFront point of presence) · **Regional** (no CloudFront, same-region clients) · **Private** (interface VPC endpoint only)
- **Stages:** most configuration is **per stage**. Each stage has its own endpoint URL and settings. A **deployment** is an immutable snapshot with an ID; a **stage** is a named pointer to one deployment. Stage names are arbitrary labels — nothing makes `prod` special
- **Canary:** enable it on a stage, and every new deployment lands on the **canary, not the base**. Configurable percentage split, adjustable at any time, **promote** to make it the new base or remove it to revert. REST APIs only
- **Caching:** **per stage**, default TTL **300 s**, range **0 to 3,600 s**, size **0.5 GB to 237 GB**, **can be encrypted**. Backends are called only on a **cache miss**. `GET` only by default; max cacheable response 1 MB; REST APIs only
- **Throttling:** account default **10,000 RPS per region** with a **5,000 burst**; per-stage/per-method throttles and **usage plans plus API keys** for per-consumer limits. Exceeding one gives a **429**
- **Integration timeout:** **29 seconds** by default, producing a **504**. Since June 2024 it can be raised beyond 29 s for **Regional and private REST APIs** via Service Quotas, possibly at the cost of a lower throttle quota — but 29 s is still the exam answer
- **Monitoring:** [CloudWatch](../services/cloudwatch.md) logs and metrics on both the client and integration sides

| If the question says...                                                                                            | Think...                                                                      |
| ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| "cache per…"                                                                                                       | **Per stage** (TTL 300 s default, max 3,600 s, 0.5 GB to 237 GB, encryptable) |
| "validate a Cognito token"                                                                                         | **Cognito user pools** — native validation                                    |
| "custom / bearer token / external identity provider"                                                               | **Lambda authorizer** (returns an IAM policy plus a principal)                |
| "credentials in headers"                                                                                           | **IAM** authentication                                                        |
| "nearest CloudFront point of presence"                                                                             | **Edge-optimized** endpoint                                                   |
| "only inside a VPC / interface endpoint"                                                                           | **Private** endpoint                                                          |
| "same region, low overhead"                                                                                        | **Regional** endpoint                                                         |
| "gradually shift a percentage of traffic to a new version"                                                         | **Canary** deployment on a stage                                              |
| "redeploy while canary is enabled"                                                                                 | Lands on the **canary, not the base**                                         |
| **429**                                                                                                            | **Throttling**                                                                |
| **502**                                                                                                            | Backend returned **bad or invalid output**                                    |
| **503**                                                                                                            | Backend **offline** / service unavailable                                     |
| **504**                                                                                                            | **Integration timeout** (29-second default)                                   |
| **403**                                                                                                            | **Access denied** (authorizer or WAF)                                         |
| "write to [DynamoDB](../services/dynamodb.md) / start a Step Functions execution / publish to SNS with no compute" | **Direct AWS service integration**                                            |
| "consistent front end during a migration"                                                                          | API Gateway as a **migration façade**                                         |

## Amazon AppFlow

See [Amazon AppFlow](../services/appflow.md) for the diagram and walkthrough, and [Data Integration](../concepts/data-integration.md#choosing-a-data-movement-service) for service selection.

> [!tip] Exam Tip
> Supported SaaS records → [S3](../services/s3.md)/Redshift with minimal custom code suggests AppFlow. Mapping, masking, filtering, and validation can be configured in a flow. A reusable connection supplies access settings; a flow defines the transfer.

| Scenario                                                     | Choice or qualification                                                                                                                                                                                                                      |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Supported SaaS transfer must avoid the public internet       | AppFlow + [PrivateLink](../services/vpc.md#aws-privatelink), if the connector and SaaS account support it; console metadata calls still use public endpoints                                                                                 |
| Run when source SaaS records change                          | Event-triggered flow, only for sources that expose supported change events                                                                                                                                                                   |
| Scheduled incremental transfer                               | Records since the last successful run; a creation timestamp alone does not capture edits                                                                                                                                                     |
| No suitable built-in connector                               | Check the [supported applications](https://docs.aws.amazon.com/appflow/latest/userguide/app-specific.html) first; a Java/Python custom connector requires a feasibility check and implementation, not guaranteed support for any application |
| Land records in S3 and register their schema                 | AppFlow can populate the [Glue Data Catalog](../services/glue.md#data-catalog-and-crawlers) for configured output; a crawler is not mandatory                                                                                                |
| Database migration / CDC rather than SaaS connector transfer | [DMS](../services/dms.md); its targets are not limited to databases                                                                                                                                                                          |
| Event matching and routing                                   | [EventBridge](../services/eventbridge.md); it can also receive supported events through AppFlow                                                                                                                                              |
| Retained streaming records / managed stream delivery         | [Kinesis Data Streams / Data Firehose](../services/kinesis.md) respectively                                                                                                                                                                  |

> [!warning] Gotcha
> Do not memorize “100 GB per flow” as a universal allowance. Connector-specific size, frequency, and application API quotas still apply. Glue also has external/SaaS connectors: the distinction is the work required, not merely whether the source is outside AWS.

## AWS Glue

See [AWS Glue](../services/glue.md) for the diagram and current service details, and [Data Integration](../concepts/data-integration.md) for the relationship between storage, catalogs, ETL, and queries.

> [!tip] Exam Tip
> **Crawler discovers schemas; Data Catalog stores metadata; job transforms records.** The catalog does not contain the underlying dataset, and publishing metadata does not grant data access.

| Question trigger                                                                                                                                                           | Answer or distinction                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Serverless ETL, cleaning, restructuring, or enriching datasets                                                                                                             | **Glue job**                                                                                                                                   |
| Automatically discover schemas in [S3](../services/s3.md), [RDS](../services/rds.md), or [DynamoDB](../services/dynamodb.md)                                               | **Glue crawler** → Data Catalog                                                                                                                |
| Shared table definitions for Athena, Redshift Spectrum, or EMR                                                                                                             | **Glue Data Catalog**                                                                                                                          |
| New S3 partition absent from Athena results                                                                                                                                | Check partition metadata; crawl again, explicitly add the partition, or use `MSCK REPAIR TABLE` for Hive-style paths                           |
| Reduce scanned data in a partitioned Athena query                                                                                                                          | Filter on **partition keys**; see the [lease-record example](../services/glue.md#worked-example--s3-lease-records-to-shared-catalog-consumers) |
| Athena partition projection enabled; other engines cannot find new partitions                                                                                              | Projection computes partitions for Athena; it does not register them for Glue, Spectrum, or EMR                                                |
| How many catalogs?                                                                                                                                                         | One **default/root catalog per account per Region**; nested/federated catalogs are also supported                                              |
| Process [Kinesis Data Streams](../services/kinesis.md#kinesis-data-streams) or [Kafka](../concepts/streaming-vs-messaging.md#kinesis-vs-kafka-and-amazon-msk) continuously | **Glue streaming ETL**, with checkpoints; crawlers do not crawl streams                                                                        |
| Write transformed data into [DynamoDB](../services/dynamodb.md)                                                                                                            | Supported by the Glue [DynamoDB](../services/dynamodb.md) writer; it is not source-only                                                        |
| Run ETL on a cron schedule                                                                                                                                                 | Glue `SCHEDULED` trigger; [EventBridge](../services/eventbridge.md) is not mandatory                                                           |
| Start an ETL workflow when data arrives                                                                                                                                    | EventBridge → Glue workflow with an `EVENT` trigger                                                                                            |
| Coordinate a Glue job with several AWS services and wait for it                                                                                                            | [Step Functions](../services/step-functions.md) Standard workflow with Glue `.sync` integration                                                |
| Ad hoc SQL over S3, without a data-preparation requirement                                                                                                                 | **[Athena](../services/athena.md)**; “ad hoc” alone does not imply Glue                                                                        |
| Glue versus AWS Data Pipeline for managed ETL                                                                                                                              | Glue; Data Pipeline is in maintenance mode and closed to new customers since July 25, 2024                                                     |
| No servers to manage means no permissions/networking work                                                                                                                  | Incorrect: configure [IAM](../services/iam.md) and [VPC](../services/vpc.md) access as required                                                |

> [!warning] Gotcha
> Serverless is a management model, not a promise of zero idle cost. Streaming Glue jobs consume capacity continuously; crawlers and catalog usage have billing separate from ETL jobs.

## Amazon Athena

See [Athena](../services/athena.md) for the schema-on-read diagram, pricing models, workgroups, and federated query.

- **Serverless, interactive SQL over data in [S3](../services/s3.md).** No cluster, no database to load, no ETL required before the first query. Athena also offers an [Apache Spark engine](../services/athena-spark.md) for PySpark notebooks; the exam focuses on SQL.
- **Schema-on-read.** Source data stays in S3 **unchanged**; table definitions in the [Glue Data Catalog](../services/glue.md#data-catalog-and-crawlers) describe how to read it and are applied only at query time. Athena tables hold metadata, not rows.
- **Formats:** CSV/TSV, JSON, Ion, Avro, Parquet, ORC, Apache web logs, and native support for [CloudTrail](../services/cloudtrail.md), [VPC Flow Logs](../services/vpc-flow-logs.md), [ELB](../services/elb.md) access logs, and Cost and Usage Reports.
- **Pricing:** per TB **scanned** (10 MB minimum per query, no base cost), or **Capacity Reservations** billed per DPU-hour for predictable cost and dedicated capacity (4 DPU and 1-minute minimum since February 2026).
- **Cut cost and latency:** columnar **Parquet/ORC**, **compression**, **partitioning**, and **CTAS** to convert raw data; all reduce bytes scanned.
- **Workgroups** are the management unit: SQL or Spark engine, team isolation, per-query and per-workgroup data usage limits, result location and encryption, and Capacity Reservation assignment.
- **Federated query** reaches non-S3 sources (DynamoDB, RDS engines, CloudWatch Logs, and others) through Lambda or Glue-connection connectors.

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | Ad hoc SQL over S3 data, no infrastructure, pay per query | **Athena** |
> | Analyze CloudTrail, VPC Flow Logs, or ELB logs stored in S3 | **Athena** |
> | Small, known CSV schema; query it quickly | `CREATE EXTERNAL TABLE` in Athena; the definition is stored in the Glue Data Catalog |
> | Many columns or unknown schema in S3 | **Glue crawler** populates the Data Catalog, then query with Athena |
> | Reduce Athena cost or speed up queries | Parquet or ORC, compression, partitioning |
> | Limit how much data one team or one query can scan | Athena **workgroup** data usage controls |
> | Predictable Athena spend or guaranteed query capacity | **Capacity Reservations** |
> | Join S3 data with DynamoDB or RDS without moving it first | Athena **federated query** |
> | Filter inside a single S3 object | S3 Select (legacy), not Athena |
> | Already running Redshift; join warehouse tables with S3 data | [Redshift Spectrum](../services/redshift.md#querying-data-without-loading-it) |
> | Interactive PySpark notebooks over S3 data, no cluster to manage | **[Athena for Apache Spark](../services/athena-spark.md)** |
> | Scheduled production Spark ETL pipeline | **Glue** ETL jobs, not Athena Spark |

- **Athena for Apache Spark:** PySpark code in notebooks runs in a serverless Spark session on DPUs, working on in-memory DataFrames that disappear when the session ends. Pricing is **per DPU-hour only** (no per-TB scanned pricing, no Capacity Reservations). A workgroup's engine type, SQL or Spark, is **fixed at creation**.

> [!warning] Gotcha
> "Athena only queries S3" is outdated: federated query reaches other sources. And "unstructured data" in course material means parseable text such as logs, not arbitrary binaries like images.

## Amazon Redshift

See [Redshift](../services/redshift.md) for the architecture diagram, node types, data protection, and Enhanced VPC Routing.

- **Petabyte-scale MPP data warehouse.** Data from many operational databases is loaded in for long-term reporting and analytics. It is **OLAP** and **column-based**, not an OLTP database like [RDS](../services/rds.md) or [Aurora](../services/aurora.md).
- **Two modes, same engine and SQL.** **Provisioned**: you choose node type and count; a **leader node** (not billed) plans queries and aggregates results, and **compute nodes** run the work in parallel across **slices**. **Serverless**: capacity in **RPUs** with a base (4 to 1024, depending on Region) and an optional maximum; a **namespace** holds data and users, a **workgroup** holds compute and the endpoint.
- **Availability.** A provisioned cluster is **single-AZ by default**; **Multi-AZ** is optional on RA3 and RG and can't be paused. Serverless needs 2 subnets in 2 AZs (3 in 3 with Enhanced VPC Routing), which is placement, not Multi-AZ.
- **Billing.** Provisioned is per node-hour while running, whether used or not; **pause** stops compute charges, and **reserved nodes** suit steady use. Serverless is per RPU-second used, with no idle compute charge; managed storage remains billable.
- **Query in place.** **Spectrum** queries [S3](../services/s3.md) without loading; **federated query** reads live RDS and Aurora PostgreSQL or MySQL data. Both still need a Redshift warehouse.
- **Access and security.** JDBC or ODBC, and native AWS integrations such as Quick Sight (formerly QuickSight). Runs in a [VPC](../services/vpc.md), private by default; [IAM](../services/iam.md) for AWS API/resource access, SQL grants for database objects; encrypted at rest by default ([KMS](../services/kms.md) key optional); [CloudWatch](../services/cloudwatch.md) monitoring.
- **Data protection.** Automatic snapshots to S3 about every **8 hours or 5 GB per node** of change, retained 1 day by default (up to 35); manual snapshots retained indefinitely by default, with configurable expiration; **cross-Region snapshot copy** for DR. Serverless uses **recovery points** (about every 30 minutes, kept 24 hours).
- **Loading.** `COPY` from S3 (`UNLOAD` back to S3), `COPY` from [DynamoDB](../services/dynamodb.md), [Firehose](../services/kinesis.md#kinesis-data-firehose-now-amazon-data-firehose) (via S3 plus `COPY`), [DMS](../services/dms.md), and [Glue](../services/glue.md) ETL.
- **Enhanced VPC Routing.** By default `COPY` and `UNLOAD` traffic goes over the internet; this feature forces it through the VPC so security groups, NACLs, endpoints, and VPC Flow Logs apply.

> [!tip] Exam Tip
>
> | Requirement | Candidate |
> | --- | --- |
> | Data warehouse, OLAP, BI reporting across data from many databases | **Redshift** |
> | Transactional application database | RDS or Aurora, not Redshift |
> | Warehouse for variable or intermittent use, no capacity management | **Redshift Serverless** |
> | Steady 24/7 warehouse at lowest cost | Provisioned with **reserved nodes** |
> | Stop paying for an idle provisioned cluster overnight | **Pause** the cluster |
> | Survive an AZ failure (provisioned) | **Multi-AZ** on RA3 or RG |
> | Redshift DR into another Region | **Cross-Region snapshot copy** |
> | Query S3 data from an existing warehouse without loading it | **Redshift Spectrum** |
> | Query live RDS or Aurora data from Redshift | **Federated query** |
> | Ad hoc SQL over S3 with no warehouse | [Athena](../services/athena.md), not Redshift |
> | Keep `COPY`/`UNLOAD` traffic in the VPC, or monitor it with Flow Logs | **Enhanced VPC Routing** |

> [!warning] Gotcha
> A default provisioned cluster is single-AZ. The course's "data is mirrored to another node" describes node-storage clusters (DC2); on RA3 and RG the durable copy lives in S3-backed Redshift Managed Storage.

## Kinesis

See [Kinesis](../services/kinesis.md) for full details, and [Streaming vs Messaging](../concepts/streaming-vs-messaging.md) for the Kinesis-vs-SQS and Kinesis-vs-Kafka decisions.

- **The family:** **Data Streams** (ingest at scale, many independent consumers, persistence window) · **Firehose** (managed *delivery* to destinations) · **Data Analytics** (real-time SQL processing) · **Video Streams** (live media ingestion). All are **public, regional, highly available** services — not placed in a VPC
- **The stream** is the base entity: producers put records in, consumers read them out, capacity comes from **shards**
- **Per shard:** **1 MB/s or 1,000 records/s in**, **2 MB/s or 2,000 records/s out**. Record size **1 MB** (raisable to 10 MiB in current AWS, but 1 MB is the exam answer). A **partition key** is hashed to pick the shard
- **Persistence window:** **24 hours by default, up to 365 days** for extra cost. This is what enables **replay** — and it is the sharpest distinction from [SQS](../services/sqs.md), where a processed message is gone
- **Enhanced fan-out:** classic consumers **share** the 2 MB/s per shard; enhanced fan-out gives **each registered consumer its own 2 MB/s per shard**, pushed over HTTP/2 with `SubscribeToShard` instead of polled. 20 consumers per stream (50 on On-demand Advantage)
- **Kinesis vs SQS:** ingestion, analytics, dashboards, IoT, clickstreams, many producers, many independent consumers, replay means **Kinesis**. Decoupling, worker pools, async, process-once-then-delete means **[SQS](../services/sqs.md)**
- **Firehose is NOT real-time — it is near real-time.** It buffers and delivers when the buffer size fills or the buffer interval passes (course figures: **1 MB or 60 s**). A question demanding real-time delivery rules Firehose out
- **Firehose destinations (memorize):** **S3**, **Redshift** (via an S3 intermediate plus a Redshift `COPY`), **Elasticsearch/OpenSearch**, **Splunk**, **HTTP endpoints**. It can transform in flight with a **[Lambda](../services/lambda.md)** function and keep unmodified source records in a backup S3 bucket
- **Firehose sources:** direct PUT, a Kinesis Data Stream (Firehose reads it as a consumer), [CloudWatch](../services/cloudwatch.md) Logs and Events, IoT, KPL, the Kinesis Agent
- **Data Analytics:** ingests from Data Streams or Firehose, optionally enriched with **static reference data from [S3](../services/s3.md)**; outputs to Firehose (**near** real-time) or to Lambda / Data Streams (**real-time**). Use it for **complex** real-time processing — Firehose plus Lambda covers only simple transforms
- **Video Streams:** live video plus non-video time-serialized media (audio, thermal, depth, radar); integrates with **Rekognition** and **Connect**. You **cannot** read the raw media directly off S3, EBS, or EFS — access goes through the product
- **Course vs reality:** Firehose is now **Amazon Data Firehose** (Feb 2024); Data Analytics for Flink is **Amazon Managed Service for Apache Flink** (Aug 2023) and the **SQL variant was discontinued on 27 January 2026**; "Elasticsearch" is now **OpenSearch Service**. The old names are still the likely exam wording
- **Kafka on AWS is [Amazon MSK](../concepts/streaming-vs-messaging.md#kinesis-vs-kafka-and-amazon-msk).** A stream is a Kafka **topic**, a shard is a **partition**, a sequence number is an **offset**. MSK brokers run **in your VPC** (Kinesis is a public regional endpoint), retention can be effectively **unlimited** with tiered storage, and **log compaction** and broker **transactions/exactly-once** have no Kinesis equivalent. **Mention of Apache Kafka, existing Kafka apps, or Kafka APIs means MSK**; "fully managed, no Kafka expertise, AWS-native producers and consumers" means Kinesis. **MSK Serverless** removes capacity planning (200 MBps in / 400 MBps out per cluster) — provisioned MSK still has you sizing brokers

| If the question says...                                                     | Think...                                                         |
| --------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| "ingest from thousands of devices, sensors, or clickstreams"                | **Kinesis Data Streams**                                         |
| "decouple", "worker tier", "async", "process once then delete"              | **[SQS](../services/sqs.md)**                                    |
| "multiple consumers must replay or re-read the same data"                   | **Kinesis Data Streams** (rolling window)                        |
| "real-time delivery to S3"                                                  | **Not Firehose** — Data Streams plus a Lambda consumer           |
| "near real-time delivery to S3 / Redshift / Splunk, nothing to build"       | **Firehose**                                                     |
| "load streaming data into Redshift"                                         | **Firehose** (S3 intermediate plus `COPY`)                       |
| "persist stream data past the window / archive to a data lake"              | **Firehose**                                                     |
| "complex real-time SQL processing of a stream"                              | **Data Analytics** (now Managed Flink)                           |
| "enrich a stream with static reference data"                                | **Data Analytics** plus an S3 reference table                    |
| "live video", "RTSP", "GStreamer", drone or camera feeds                    | **Kinesis Video Streams** (plus Rekognition)                     |
| "each consumer needs the full 2 MB/s with no contention"                    | **Enhanced fan-out**                                             |
| "one shard throttles while the others idle"                                 | **Partition key** design                                         |
| "read the raw video from the S3 bucket"                                     | **Wrong** — Video Streams data is only reachable via the product |
| "existing Apache Kafka applications", "Kafka APIs", "already running Kafka" | **Amazon MSK**                                                   |
| "Kafka Connect / Kafka Streams / ksqlDB / schema registry"                  | **Amazon MSK**                                                   |
| "retention beyond 365 days" or "keep the latest value per key forever"      | **Kafka/MSK** (tiered storage, log compaction) — not Kinesis     |
| "Kafka, but no capacity planning"                                           | **MSK Serverless**                                               |
| "streaming brokers must sit inside the VPC"                                 | **MSK** (Kinesis is a public endpoint)                           |

## DNS and Route 53

See [Route 53](../services/route53.md) and [DNS Concepts](../concepts/dns.md) for full details.

- Route 53 is an **authoritative DNS** service — **globally resilient**, name servers span multiple regions
- Each hosted zone gets **4 public name servers**; NS records at the registrar must point to them
- **Alias record**: Route 53-specific; works at the zone apex (`example.com`); CNAME cannot; **no query charge** to AWS resources
- CNAME only works for **subdomains** — never for the root domain
- Alias type must **match the target resource's record type** (e.g. ELB returns A → create A Alias)
- **Private Hosted Zone**: resolves only within a VPC; requires `enableDnsSupport` and `enableDnsHostnames` on the VPC
- **Cross-account private hosted zone association**: CLI/API only — not available in console
- **Split-View DNS**: same domain name, public + private zone → different answers based on query origin
- **PHZ namespace replacement (gotcha)**: a private zone is **not** an overlay — once it matches a query name (identical or **parent** match), the VPC Resolver answers **only** from that zone. A missing record returns **NXDOMAIN**, *not* the public answer. So an apex PHZ for `example.com` must duplicate **every** public name still needed inside the VPC (e.g. `customer.example.com`), or those names go dark internally. Public fall-through happens **only when no associated PHZ owns the namespace**, never per-record. (Escape hatches: a Resolver rule for the same domain **overrides** the PHZ; NS records in the PHZ can delegate a subdomain.) See [How association and namespace resolution work](../services/route53.md#how-association-and-namespace-resolution-work).
- TTL = how long resolvers cache a record; **lower TTL before migrations**, raise after

### How DNS Resolves (Walking the Tree)

See [DNS — Resolution Chain](../concepts/dns.md#resolution-chain) for the full walk and diagrams.

- DNS's whole job is to **find the one zone** that can give an authoritative answer; the climb from root down to the domain is called **walking the tree**, and **every query hands you the next step** (a referral *is* the NS records for the tier below)
- The recursive **resolver** (home router or ISP) does the walk on your behalf and **caches** the result; the client only ever talks to the resolver

| Trigger phrase                                                   | Answer                                                          |
| ---------------------------------------------------------------- | --------------------------------------------------------------- |
| "static name-to-IP mapping that overrides DNS, checked first"    | **hosts file** (`/etc/hosts`)                                   |
| "queries on your behalf, runs on a router or ISP, caches"        | **recursive resolver**                                          |
| "hard-coded IPs maintained by the OS vendor, not looked up"      | **root servers**                                                |
| "points `.com` at VeriSign / points a domain at its nameservers" | **NS records = delegation**                                     |
| "the definitive answer from the source-of-truth nameserver"      | **authoritative**                                               |
| "a cached copy at a resolver, may be stale"                      | **non-authoritative** (resolvers always label answers this way) |

> [!warning] Gotcha
> An authoritative answer is **not always an IP** — it can be a **CNAME** (another name), forcing the resolver to **walk the tree again** for that name. Chained CNAMEs stack latency and are a real cause of slow apps. And the **hosts file overrides DNS**: it's checked before any query, so a stale entry there is the classic *"why isn't my DNS change working?"* trap.

### Registrar vs DNS Hosting (Interoperability)

See [Registrar vs DNS Hosting](../services/route53.md#registrar-vs-dns-hosting-interoperability) for full details.

- "Registering a domain" is **two independent roles**, each can live with a different provider:
  - **Registrar** = registers the domain with the TLD **registry**; **one-off per-year** (or per-3-year) fee; console **Registered Domains**
  - **DNS Hosting** = the **public hosted zone** (4 name servers + zone file); **monthly** fee; console **Public Hosted Zone**
- **Registry ≠ Registrar**: the registry *owns* the TLD zone (`.org` = **PIR**); the registrar *registers with* it on your behalf
- **NS delegation records** are the handshake — the registry always points the domain at its 4 name servers; present in every configuration
- Three valid arrangements:
  - **Both via R53** — the default
  - **R53 registrar only** (3rd party hosts) — valid but **rare/discouraged**; you lose R53's main value (its DNS hosting)
  - **R53 hosting only** (3rd party registers) — **common**; create a public hosted zone in R53, give the 4 NS to the external registrar

> [!tip] Exam Tip
> **Migrating an old domain registered elsewhere to R53 DNS** = R53 **hosting only**, done *after the fact*: create a public hosted zone in R53 and update the **NS records at the existing registrar** to the 4 R53 name servers. No re-registration. Triggers: "domain registered with GoDaddy/Hover but we want R53 to manage records", "keep the existing registrar".

> [!warning] Gotcha
> Registration fee (per-year, to the registrar) and hosting fee (monthly, per hosted zone, to the host) are **separate fees**. And R53 registrar-only is the trap answer when a question implies it delivers R53's core benefit — it doesn't; the value is in **hosting**.

### DNS Record Types

See [DNS — Record Types](../concepts/dns.md#dns-record-types) for full details and diagrams.

| Record    | Maps / does                                                     | Trigger keywords                                             |
| --------- | --------------------------------------------------------------- | ------------------------------------------------------------ |
| **NS**    | Delegation — parent zone points at the child zone's nameservers | "delegation", "points at the servers hosting the zone"       |
| **A**     | Hostname to **IPv4**                                            | "hostname to IPv4"                                           |
| **AAAA**  | Hostname to **IPv6**                                            | "hostname to IPv6"                                           |
| **CNAME** | Host to **another host (name)**                                 | "alias", "shortcut", "reduce admin overhead" — ⚠️ never an IP |
| **MX**    | Find the **mail server (SMTP)** for a domain                    | "mail server", "email delivery"                              |
| **TXT**   | Arbitrary text on a domain                                      | "prove domain ownership", "verification", "SPF / anti-spam"  |

- **CNAME → name only, never an IP**, and never at the **zone apex** — use a Route 53 **Alias** for the apex case
- **MX: lower priority number = higher priority** (tried first). A value with **no trailing dot** is a host inside the same zone (`mail` means `mail.example.com`); a value **with a trailing dot** is an FQDN that can point to an external provider (Office 365, etc.). The host name itself is arbitrary
- **TXT** is the answer for **domain-ownership verification** (the external service hands you a string to publish) and **SPF/anti-spam** sender authorization
- **NS records are delegation** at every level (root → TLD → domain) — the answer whenever a question describes a parent zone pointing at the servers that host a child zone
- **Authoritative** answer = from the domain's source-of-truth nameserver (slower, always current); **non-authoritative** = a cached copy at a resolver (faster, can be stale until TTL expires)

> [!warning] Gotcha
> Two record-type traps to lock in: **(1)** a CNAME can only point at another name — if an answer has a CNAME pointing straight at an IP, it's wrong. **(2)** MX priority is "lowest number wins", not highest. And the TTL trap: migrating email with a **high MX TTL** leaves old mail servers cached and breaks delivery — **lower the TTL days in advance** of any DNS migration.

### Route 53 Routing Policies

| Policy       | When to use                                                     | Health checks?       |
| ------------ | --------------------------------------------------------------- | -------------------- |
| Simple       | Single endpoint                                                 | No                   |
| Weighted     | A/B testing, gradual traffic shifts, simple load balancing      | Optional             |
| Latency      | Route to lowest-latency AWS region (performance)                | Optional             |
| Failover     | Active/passive HA; primary + secondary                          | Required on primary  |
| Geolocation  | Restrict/route by user's state/country/continent                | Optional             |
| Geoproximity | Closest resource by physical distance, tunable with bias        | Optional             |
| Multi-value  | Return multiple IPs; improve availability (not a load balancer) | Optional, per record |

- **Weighted**: each same-name record gets a weight; returned with probability `weight ÷ total weight` — weights **need not sum to 100**. Weight **0** = never returned, **unless all are 0** (then all eligible). Health checks **don't change the denominator**: an unhealthy selected record is skipped and the draw repeats; its share is **not** redistributed. Triggers: "percentage of traffic", "test a new version", "send 5 % to the new build".
- **Latency**: each record tagged with an **AWS region** (one record per name per region); AWS holds a **latency database** between user location and regions. Returns the **lowest-estimated-latency healthy** record; if unhealthy, the next-lowest. Latency data is **estimated / not real-time**. Triggers: "lowest latency", "best performance for a global app".
- **Geolocation vs Geoproximity** — the most-tested distinction:
  - **Geolocation** = match by **tagged location** (US state, country, continent, default). Specificity order State → Country → Continent → Default, stopping at the first match; no match and no default → **"no answer"**. It returns **relevant records only, NEVER the closest** as a fallback. Triggers: "restrict content by region", "block/allow by country", "language-specific content".
  - **Geoproximity** = **physical distance** to the resource (AWS region for AWS resources, **lat/long** for external), plus a **bias**: `+` grows a resource's service area (shrinking neighbours), `-` shrinks it. Only geoproximity lets you **shift traffic away from the geographically closest** resource. Triggers: "closest resource", "bias", "route more/less traffic to a region".
- **Multi-value**: returns **up to 8 healthy** records (random 8 if more exist), each with its **own health check**; client picks one. **Improves availability, is NOT a load balancer** — it doesn't manage connections.

> [!tip] Exam Tip
> **Failover routing** requires a Route 53 **health check** on the primary record. If the primary fails the health check, Route 53 automatically returns the secondary record. It uses **two records of the same name** (one Primary, one Secondary). Trigger phrases: "active/passive", "primary and secondary", "fail over to a backup/maintenance page". The classic pattern is **out-of-band failover** — primary and secondary point at *different* services (e.g. an EC2 primary with an S3 static-page secondary) so the backup survives a platform-level failure of the primary. Don't confuse with **weighted/multi-value**, which *distribute* across healthy resources rather than active/passive switching.

> [!warning] Gotcha
> Two perennial routing traps: (1) **geolocation never returns the nearest record** — only a location-match, the default, or "no answer"; if a question says "closest", that's **geoproximity** or **latency**, not geolocation. (2) **Multi-value is not a load balancer** — it improves availability via DNS but performs no connection management; true load balancing means an **ELB**.

### Route 53 Health Checks

See [Route 53 Health Checks](../services/route53.md#health-checks) for full details.

- Health checks are **separate objects** from DNS records — created under Route 53 → Health Checks, then attached to records
- Can monitor **any public endpoint** by IP or domain — not just AWS resources
- Three categories: **Endpoint** (direct probe), **CloudWatch Alarm** (any metric), **Calculated** (boolean of other checks)
- Endpoint probe types: **TCP** (handshake in 10 s), **HTTP/HTTPS** (status 200–399), **HTTP/HTTPS + string match** (matches first 5,120 bytes of body)
- **String matching is the most accurate** application health signal — catches "200 OK but app is broken" cases
- Default interval **30 s**; fast mode **10 s** at extra cost
- **18 % rule**: if more than 18 % of distributed checkers report healthy → overall healthy; otherwise unhealthy
- Firewalls ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) must **allow Route 53 health checker IP ranges** or checks will false-fail
- Unhealthy records are normally **excluded from DNS responses**
- **SNS integration** for alerting on state change — independent of routing behaviour

> [!tip] Exam Tip
> When a question describes verifying that an app is **actually working** (not just reachable), the answer is **HTTP/HTTPS with string matching**. TCP only proves the port is open; plain HTTP only proves the web server responds; string matching proves the response body is correct.

> [!warning] Gotcha
> Health checks live **outside hosted zones**. Look for them under Route 53 → Health Checks in the console, not inside the record set.

### DNSSEC

See [DNSSEC](../concepts/dnssec.md) for the full mechanics, worked example, and `dig` commands.

- **DNSSEC** is a *secure add-on* to DNS (not a replacement): it adds **origin authentication** + **data integrity** via digital signatures, building a **chain of trust** from the root down. It is **additive and backward compatible** — DNS-only clients never see the extra records. The problem it solves is **cache poisoning**
- It **validates**, it does **not correct** — it only tells you whether data is authentic/unmodified, never what the right answer should be
- **DNSSEC validates RRSETs** (all records with the *same name + same type*), **not individual records** — a classic trap
- Key/record vocabulary: **RRSIG** = signature of an RRSET (made by the private **ZSK**); **DNSKEY** = publishes the public keys (**flag 256 = ZSK, 257 = KSK**); **ZSK** signs all zone RRSETs; **KSK** signs *only the DNSKEY RRSET* and is the zone's point of trust; **DS** record in the **parent** holds a **hash of the child's KSK** (the cross-zone trust link)
- The **root KSK has no parent**, so it is a **trust anchor** — explicitly trusted by every validating resolver, distributed out-of-band by IANA, and re-signed at the quarterly **root signing ceremony** (HSMs at El Segundo / Culpeper, Crypto Officers, Verisign as root-zone maintainer)
- Four validation outcomes: **Secure** (chain verified, sets the **AD** flag) · **Insecure** (proven gap, e.g. unsigned zone with no DS) · **Bogus** (signatures fail → **SERVFAIL**) · **Indeterminate** (no trust anchor reachable)
- **On AWS**: [Route 53](../services/route53.md) does **DNSSEC signing** for public hosted zones (you bring an asymmetric **[KMS](../services/kms.md)** key as the KSK in `us-east-1`; Route 53 manages the ZSK; you must add the **DS record to the parent** to complete the chain), and **Route 53 Resolver** can do **DNSSEC validation** per-VPC

Route 53 signing **implementation** specifics (see [Implementing DNSSEC](../services/route53.md#dnssec)):

- The enabling sequence **starts with KMS** (asymmetric key pair backing the KSK), then Route 53 makes the ZSK, publishes the DNSKEY, signs it with the private KSK (`RRSIG DNSKEY`), then you add the parent DS and the CloudWatch alarms
- The KSK's KMS key **must be in `us-east-1`** — a hard requirement regardless of where the zone serves traffic
- **ZSK creation *and rotation* are handled internally by Route 53 — KMS is not involved in the ZSK** (KMS backs only the KSK). This KSK↔KMS / ZSK↔R53-internal split is a frequent exam distinction
- **DS record to the parent**: if the domain is **registered with Route 53**, add it from the **Registered Domains** console (R53 liaises with the TLD); if registered **elsewhere**, you add it **manually** at that registrar
- Create CloudWatch alarms **`DNSSECInternalFailure`** and **`DNSSECKeySigningKeysNeedingAction`** — both flag an urgent KSK/KMS-level problem with the zone
- Optional **per-VPC validation**: on a DNSSEC-enabled zone, a record that fails validation is **not returned** (non-DNSSEC zones are unaffected)

| Trigger phrase                                    | Answer                                                               |
| ------------------------------------------------- | -------------------------------------------------------------------- |
| "validate DNS data / detect tampered DNS answers" | **DNSSEC**                                                           |
| "protect against cache poisoning / spoofed DNS"   | **DNSSEC**                                                           |
| "all records with the same name and type"         | **RRSET** (what DNSSEC signs)                                        |
| "signature of a record set"                       | **RRSIG** (made by the private ZSK)                                  |
| "parent holds a hash of the child's key"          | **DS record** (hash of the child **KSK**)                            |
| "root key, no parent, explicitly trusted"         | **trust anchor** (root KSK)                                          |
| "sign a public hosted zone in AWS"                | **Route 53 DNSSEC signing** + a **KMS** KSK + a parent **DS** record |
| "where must the KMS key for the KSK live?"        | **`us-east-1`**                                                      |
| "who creates/rotates the ZSK in Route 53?"        | **Route 53 internally** (KMS not involved)                           |
| "alarms to set when enabling DNSSEC"              | **`DNSSECInternalFailure`**, **`DNSSECKeySigningKeysNeedingAction`** |

> [!warning] DNSSEC Gotchas
>
> - It **validates, never corrects** — and it signs **RRSETs, not individual records**.
> - The **private ZSK is never in the zone**; only the **public** ZSK (via DNSKEY) is. The **DS** record holds a **hash of the KSK**, not the ZSK and not the full key.
> - **Bogus ≠ Insecure**: Bogus means signatures are present but *fail* (returns SERVFAIL); Insecure means a *proven* absence of signing (no DS). "No RRSIG" can mean an unsigned zone (Insecure), a signed **NSEC3** proof of non-existence (still **Secure**), or a CNAME into an unsigned zone (Insecure leaf) — not necessarily a failure.
> - Route 53: enable/disable has a strict **order** (sign before adding the parent DS; remove the DS and wait its TTL before unsigning) or validating resolvers SERVFAIL the whole zone.

## AWS Organizations

See [Organizations](../services/organizations.md) for full details.

- **Management account** (formerly master) creates the org and is the **payer** for all member accounts
- **Member accounts** lose their individual billing — single consolidated bill
- **OUs** allow nested grouping (root → OU → sub-OU → account); SCPs attach at root/OU/account and inherit downward
- **Volume discounts + RI/SP benefits pool org-wide by default** — bought-in-account-A floats to account-B's matching usage
- New accounts created inside the org get an automatic **`OrganizationAccountAccessRole`** trusting the management account → one-click Switch Role

### SCPs

See [Service Control Policies](../services/organizations.md#service-control-policies-scps) for full details.

- **SCPs are guardrails, not grants** — they set the **maximum** permissions; a principal still needs an IAM identity policy that *allows* the action
- **Effective permissions = SCP allows ∩ IAM identity policy allows** — the intersection. Anything outside the overlap is denied
- **SCPs do NOT apply to the management account** — even `Deny *` is ignored. Keep workloads in member accounts so SCPs actually take effect
- SCPs **can** indirectly restrict the **account root user** — one of the few mechanisms in AWS that bounds root
- Default model is **deny list**: AWS attaches a `FullAWSAccess` SCP everywhere; you add explicit `Deny` statements for what you don't want. Low overhead, new AWS services are automatically permitted
- **Allow list** model = remove `FullAWSAccess`, add explicit `Allow`s. More secure, much higher admin overhead — every service must be added

> [!tip] Exam Tip
> "Many accounts, want one bill" / "maximize RI utilization across accounts" → consolidated billing via **Organizations**. "Prevent accounts from using region X" / "block disabling CloudTrail" / "restrict EC2 instance sizes" → **SCP** at the right OU.

> [!warning] Gotcha
> A `Deny *` SCP attached to the management account does nothing — SCPs are skipped for the management account by design. If the question asks how to restrict the management account, the trick answer is "you can't with SCPs — move workloads to a member account."

### Control Tower

See [Control Tower](../services/control-tower.md) for full details.

- **Control Tower = "Organizations with superpowers"** — it **orchestrates** other services (Organizations, IAM Identity Center, CloudFormation, Config, CloudWatch, CloudTrail, SNS, Service Catalog) to stand up a well-architected multi-account environment. It does not reinvent them
- **Four parts:** **Landing Zone** (the environment), **Guardrails/Controls** (governance rules), **Account Factory** (automated account provisioning), **Dashboard** (oversight)
- **Landing Zone** has a **home region** that is **always available and cannot be denied**; other regions can be allowed/denied. Default OUs: **Foundational (Security)** holding the **Audit** + **Log Archive** accounts, and **Custom (Sandbox)**
- **Log Archive account** = isolated, **read-only**, explicitly-granted store of Config + CloudTrail logs from **all enrolled accounts**. **Audit account** = SNS governance/security notifications + CloudWatch monitoring + home for third-party audit tools
- **Guardrails, two independent axes:** enforcement level (**mandatory / strongly recommended / elective**) and functional type (**preventative = SCP, stops the action**; **detective = Config rule, only flags it**). Optional modern third type: **proactive = CloudFormation hooks**, block at deploy time
- **Account Factory** = self-service automated account creation with guardrails auto-applied and cookie-cutter **account + network baselines** (non-overlapping VPC IPs), via console / Service Catalog / API
- **IAM Identity Center** = formerly **AWS SSO**; both federated and internal identities funnel through it for SSO

> [!tip] Exam Tip
> "Quickly set up a **secure, well-architected, multi-account** environment with governance baked in" → **Control Tower** (not raw Organizations). "Self-service standardized account provisioning with guardrails pre-applied" → **Account Factory**. "Isolated read-only log store from every account" → **Log Archive**. "Governance-change notifications / third-party audit tooling" → **Audit account**.

> [!warning] Gotcha
> **Preventative ≠ detective.** Preventative guardrails *stop* actions (SCP); detective guardrails only *identify* non-compliance after the fact ([Config rules](../services/config.md#config-rules)). Detective does not block. Also: **IAM Identity Center and AWS SSO are the same service**; **guardrails and controls are the same thing** (console rename).

## Cost Optimization

See [Cost Optimization](../patterns/cost-optimization.md) for the decision tree and pooling rules, and [EC2 Purchase Options](../patterns/ec2-purchase-options.md) for the EC2-specific deep dive.

### Purchase Plans

| Plan                                                          | Term                 | Discount                         | What it covers                                                                                                       |
| ------------------------------------------------------------- | -------------------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **On-Demand Instances**                                       | None                 | None                             | Per-second EC2 compute; no commitment, **no capacity priority**                                                      |
| **Compute Savings Plans**                                     | 1 / 3 yr             | up to ~66 %                      | EC2 (any family/Region/OS), Fargate, Lambda                                                                          |
| **EC2 Instance Savings Plans**                                | 1 / 3 yr             | up to ~72 %                      | Specific family in a specific Region (size+OS flex)                                                                  |
| **Standard Reserved Instances**                               | 1 / 3 yr             | up to ~72 %                      | Family/OS/Region locked; size flexibility only                                                                       |
| **Convertible Reserved Instances**                            | 1 / 3 yr             | up to ~54 %                      | Exchangeable for different family/OS/tenancy                                                                         |
| **Scheduled Reserved Instances** *(retired 2024-09-03)*       | 1 yr min             | Modest vs On-Demand              | Recurring time-window EC2 — **discontinued; use On-Demand Capacity Reservations**                                    |
| **On-Demand Capacity Reservations**                           | None (immediate-use) | None                             | Pure capacity guarantee in one AZ; billed even idle                                                                  |
| **Reserved Instances / Reserved Capacity** (service-specific) | 1 / 3 yr             | Varies by service and commitment | Eligible database instances or provisioned throughput; DynamoDB reserved capacity applies to provisioned mode        |
| **Database Savings Plans**                                    | 1 yr                 | Up to 35%, depending on usage    | Eligible database usage, including DynamoDB on-demand and provisioned throughput; check service-specific eligibility |
| **Spot Instances**                                            | None                 | up to ~90 %                      | Spare EC2 capacity, 2-minute interruption warning                                                                    |
| **Dedicated Instances**                                       | None                 | Premium (per-Region hourly fee)  | No co-tenant hardware; host opaque                                                                                   |
| **Dedicated Hosts**                                           | None or per-host     | Pay-per-host                     | Socket/core-based **BYOL** licensing; Host Affinity                                                                  |

> [!warning] Gotcha
> **Compute and EC2 Instance Savings Plans do not cover database usage, but Database Savings Plans cover eligible usage, including DynamoDB on-demand and provisioned throughput.** Compare service-specific reservations with Database Savings Plans; their discounts do not stack on the same usage. The older “databases never qualify for Savings Plans” shortcut is incorrect. See [AWS Savings Plans types](https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html) and [Database Savings Plans eligibility](https://aws.amazon.com/savingsplans/faqs/).

> [!warning] Gotcha — capacity priority under shortage
> If a Region or AZ runs short on EC2 capacity (e.g. after a major fault), the safe mental model is **reserved capacity first**, then ordinary On-Demand if capacity exists, then Spot only from spare capacity. A *discount* (Regional Reserved Instance, Savings Plan) gives you **no** capacity claim — only a Zonal Reserved Instance or an On-Demand Capacity Reservation does.

> [!tip] Exam Tip
> Pattern-match purchase plans: workload moves between EC2/Fargate/Lambda → **Compute Savings Plans**. Pinned `m5` in `us-east-1` → **EC2 Instance Savings Plans** (deeper discount). Long-running RDS → **Reserved DB Instance**. Batch / CI / checkpointable → **Spot Instances**. Many accounts → consolidate via **AWS Organizations** so Savings Plans / Reserved Instances pool automatically. **BYOL per-socket / per-core licensing** → **Dedicated Host** (pay per host, host targeting). **"No other customer's hardware" but don't need to target the host** → **Dedicated Instance**. **Need cheapest 24/7/365 EC2 long-term** → **3-year All-Upfront Standard Reserved Instance**. **Need a Reserved Instance but workload family/OS/tenancy may change** → **Convertible Reserved Instance** (smaller ~54% discount but exchangeable for equal/greater value). **Recurring time window, not 24/7** → historically **Scheduled Reserved Instances** (retired 2024-09-03); for current designs use **On-Demand Capacity Reservations** in the window.

### EC2 Capacity Reservations — discount vs capacity

A reservation has a **billing component** and a **capacity component**, and they are independent.

| Setup                                                           | Discount? | Capacity reserved? | Notes                                             |
| --------------------------------------------------------------- | --------- | ------------------ | ------------------------------------------------- |
| **Regional Reserved Instance** alone                            | Yes       | No                 | Flexible across AZs/sizes; no capacity guarantee  |
| **On-Demand Capacity Reservation** alone                        | No        | **Yes (one AZ)**   | Capacity guaranteed; billed at full On-Demand     |
| **Regional Reserved Instance + On-Demand Capacity Reservation** | Yes       | **Yes (one AZ)**   | The AWS-recommended combo for both benefits       |
| **Zonal Reserved Instance**                                     | Yes       | **Yes (one AZ)**   | Both in one purchase; locked to one AZ            |
| **Savings Plan + On-Demand Capacity Reservation**               | Yes       | **Yes (one AZ)**   | Same stacking trick as Regional Reserved Instance |

> [!warning] Trap — discount stacking only works with Regional Reserved Instances (or Savings Plans)
> A **Zonal Reserved Instance's discount does not apply to a separate On-Demand Capacity Reservation** — the Zonal Reserved Instance already includes its own bundled capacity reservation. The "get both discount and capacity" pattern is **Regional Reserved Instance + On-Demand Capacity Reservation** or **Savings Plan + On-Demand Capacity Reservation**.

> [!tip] Exam Tip
> When a question wants **both** a billing discount *and* a capacity guarantee, the canonical answers are: (1) buy a **Zonal Reserved Instance** if you're committed long-term and AZ-locked is fine, or (2) pair a **Regional Reserved Instance** (or Savings Plan) with an **On-Demand Capacity Reservation** if you need AZ/size flexibility plus capacity in one specific AZ.

### EC2 Purchase Options — pattern-match cheat-sheet

| Question pattern                                                                        | Answer                                                                                                                              |
| --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| "Default — short or unknown duration, cannot tolerate interruption"                     | **On-Demand Instances**                                                                                                             |
| "Cheapest possible — can resume from checkpoint / stateless / batch"                    | **Spot Instances**                                                                                                                  |
| "Spot can be interrupted when ___"                                                      | AWS needs capacity back, or Spot price rises **above your configured maximum**                                                      |
| "Two customers both running Spot Instances at different max prices — what do they pay?" | **Same current Spot price** — max only sets price tolerance                                                                         |
| "Long-running 24/7/365, lowest possible cost"                                           | **3-year All-Upfront Standard Reserved Instance**                                                                                   |
| "Long-running, but workload family/OS/tenancy might change mid-term"                    | **Convertible Reserved Instance** (~54% discount but exchangeable)                                                                  |
| "Convertible RI exchange — can I downgrade?"                                            | **No** — exchange requires equal or greater value                                                                                   |
| "Need EC2 only every Friday for 24 hours" / "recurring time window"                     | Historically Scheduled Reserved Instances — **retired 2024-09-03**; new answer is **On-Demand Capacity Reservations** in the window |
| "Guaranteed capacity in one AZ, no long-term commitment"                                | **On-Demand Capacity Reservations**                                                                                                 |
| "Discount across families and from EC2 to Fargate/Lambda"                               | **Compute Savings Plans**                                                                                                           |
| "Discount locked to `m5` in `us-east-1`, deepest possible Savings Plan"                 | **EC2 Instance Savings Plans**                                                                                                      |
| "BYOL Oracle / SQL Server / Windows per-socket licensing"                               | **Dedicated Hosts**                                                                                                                 |
| "No co-tenant hardware, don't need to target the host"                                  | **Dedicated Instances**                                                                                                             |
| "Region-wide capacity shortage — what guarantees launch?"                               | **Reserved capacity** — Zonal RI or On-Demand Capacity Reservation                                                                  |
| "Reserved Instance bought in account A, account A doesn't use it"                       | Floats automatically to a sibling account with matching usage (default org behaviour)                                               |
| "Unused Reserved Instance or Capacity Reservation costs anything?"                      | **Yes — billed whether used or not**                                                                                                |

> [!warning] Common purchase-option gotchas
>
> - **On-Demand Instances have no capacity guarantee by default.** Discount and capacity are separate — use Zonal RI or On-Demand Capacity Reservation when launchability matters
> - **"You always pay the current Spot price, not your maximum."** Higher max = more price tolerance, *not* a higher price, and it does not protect against capacity reclaim
> - **Reserved Instances are billed if idle.** Plan reservations carefully; an unused 3-year All-Upfront Reserved Instance is wasted money (mitigated within an AWS Organization by automatic floating)
> - **Convertible Reserved Instance exchanges go up, not down.** The new RI(s) must be of equal or greater value than the current one — you cannot downgrade via exchange. Useful for *shape changes* (M-family → Graviton, OS swap, Shared → Dedicated tenancy mid-term), not for shrinking commitments
> - **Scheduled Reserved Instances were retired on 2024-09-03** — no longer purchasable; AWS recommends On-Demand Capacity Reservations instead. Historical minimums were 1,200 hr/year + 1-year commitment. Source: the in-post 2024-09-03 update on the [original launch announcement](https://aws.amazon.com/blogs/aws/new-scheduled-reserved-instances/) (the product shipped with only a 5–10% discount on legacy C3/C4/M4/R3 families — narrow enough that Savings Plans and On-Demand Capacity Reservations made it redundant)
> - **On-Demand Capacity Reservations count toward your per-Region On-Demand Instance limits** even when unused
> - **Zonal Reserved Instance ≠ Regional Reserved Instance + Capacity Reservation** for discount stacking — a Zonal Reserved Instance's discount does NOT apply to a separately-purchased On-Demand Capacity Reservation
> - **"Dedicated" is two things.** Dedicated Host (own the host, BYOL per-socket) vs Dedicated Instance (no co-tenant hardware, host opaque) — see [the tenancy gotcha](../services/ec2.md#tenancy-shared-vs-dedicated)

## TLS, HTTPS, and ACM

See [TLS/HTTPS Concepts](../concepts/tls-https.md) for full details.

- **[ACM](../services/acm.md)**: non-exportable public certificates support managed integrations such as [ALB](../services/elb.md), [CloudFront](../services/cloudfront.md), and [API Gateway](../services/api-gateway.md); ACM-issued certificates can qualify for managed renewal, imported certificates do not
- **Exportable public certificates** can export their private keys for servers such as EC2; they have separate charges and require deployment of renewed copies
- **CloudFront + ACM**: the certificate **must be in `us-east-1`** (N. Virginia), regardless of distribution origin regions
- **ALB + ACM**: certificate must be in the **same region as the ALB**
- ACM DNS validation uses the supplied **CNAME**, not TXT; [Route 53](../services/route53.md) can create it automatically when permissions and zone placement allow
- **ACM Private CA**: issues private (non-publicly-trusted) certs for internal services

> [!warning] Gotcha
> **TLS termination at ALB** means ALB-to-EC2 traffic is plaintext by default. Configuring HTTPS on the target group (with a cert, possibly self-signed, on the instances) re-encrypts that hop — but it is still **hop-by-hop**, not unbroken: the ALB always terminates and re-establishes. If a question demands **unbroken** end-to-end encryption (never decrypted on the LB), the answer is an [NLB with a TCP listener](../services/elb.md#unbroken-end-to-end-encryption-tcp-pass-through). These three arrangements are the ELB [SSL connection modes](../services/elb.md#ssltls-connection-modes--bridging-pass-through-offload): bridging (terminate + re-encrypt), offload (terminate + plain HTTP), pass-through (never decrypted).

> [!tip] Exam Tip
> The TLS handshake uses **asymmetric encryption (RSA/ECDH) only for key exchange** — the actual data is encrypted with symmetric AES, which is far faster. The server's public key (from the cert) is used to securely establish a shared secret; after that, RSA is done.

## CloudWatch Logs

See [CloudWatch](../services/cloudwatch.md) for full details.

- CloudWatch Logs is a **regional, public-endpoint** service — reachable from VPCs, on-prem, and other clouds (given network + IAM access)
- Hierarchy: **Log Group → Log Stream → Log Event**; configuration (retention, permissions, metric filters) lives at the **log group** level
- **Log event** = timestamp + raw message block; CloudWatch Logs does not parse or interpret the content
- Native integrations such as [VPC Flow Logs](../services/vpc-flow-logs.md), [Lambda](../services/lambda.md), [CloudTrail](../services/cloudtrail.md), and [Route 53](../services/route53.md) require the appropriate service logging configuration and permissions; they do not require a host agent
- **Unified CloudWatch Agent** is required for OS-level logs from EC2 and on-prem servers — EC2 does **not** auto-send logs, only basic metrics
- **Metric filters** scan log content for patterns → increment a CloudWatch Metric → optionally trigger an alarm

> [!tip] Exam Tip
> Pattern-match: *"detect failed SSH attempts and alert"* → **metric filter on a log group + CloudWatch Alarm + SNS**. *"EC2 instance logs not appearing in CloudWatch"* → **CloudWatch Agent not installed or IAM role missing**.

> [!warning] Gotcha
> Log **streams** have no configuration of their own — retention, permissions, and metric filters all attach to the **log group**. If a question asks where to set retention, the answer is the log group.

### CloudWatch Agent

- Standard CloudWatch sees an instance **from the outside** (hypervisor CPU, network, disk-ops). It does **not** report in-guest **memory usage**, **disk-space-used**, per-process, or per-core detail, and it captures **no OS or application logs**
- The **CloudWatch Agent** (unified agent) runs **inside** the instance and injects OS-level **metrics** into CloudWatch and **logs** into CloudWatch Logs
- It needs **(1) a config** (which metrics + log files) and **(2) permissions via an [instance role](../services/ec2.md#instance-roles--instance-profiles)** (`CloudWatchAgentServerPolicy`) — never baked-in access keys
- **Log mapping:** one **log group per log file**, one **log stream per instance** (stream defaults to the instance ID) — *group by file, stream by instance*
- Agent OS-level metrics show up under a new **`CWAgent`** namespace — its presence is the fingerprint the agent is installed
- Store the agent config in **[SSM Parameter Store](../services/ssm.md#parameter-store)** once and reuse it across a fleet (pairs with CloudFormation + Auto Scaling); other deploy paths: Systems Manager, AMI baking, user-data bootstrapping

> [!tip] Exam Tip
> Pattern-match: *"alarm on EC2 memory / disk-space used"* → **CloudWatch Agent** (custom metric); standard CloudWatch can't. *"collect /var/log/… or application logs from EC2"* → **CloudWatch Agent**. *"new `CWAgent` namespace appeared"* → the agent is installed. *"agent installed but a specific log never appears"* → **file-permission mismatch** between the log and the agent's run-as user (`cwagent`), not IAM.

> [!warning] Gotcha — run-as user vs IAM
> IAM (the instance role) decides whether the agent can **send** to CloudWatch. **OS file permissions** decide whether its **run-as user** (`cwagent` by default; older setups use `root`) can **read** the source log. Root-only files like `/var/log/secure` are silently skipped under `cwagent`. Missing logs with everything else working → check file permissions, not the role.

## CloudTrail

See [CloudTrail](../services/cloudtrail.md) for full details.

- CloudTrail records **API / account activity** as **events** — who did what, when, from where. This is the **audit** service; [CloudWatch](../services/cloudwatch.md) is the **health/telemetry** service, and **[AWS Config](../services/config.md)** is the **resource-configuration/compliance** service
- **Event History** is on **by default, free, 90 days, management events only** — no S3 storage. For retention beyond 90 days, data events, or log delivery you must create a **trail**
- **Four event types**: **Management** (control-plane, on by default), **Data** (S3 object access, Lambda invokes — **opt-in, extra cost, high volume**), **Network activity** (supported VPC endpoint activity — opt-in, extra cost), **Insight** (anomaly detection over management-event activity — opt-in)
- **Trail scope**: **single-region** or **all-regions** (the latter auto-adds new regions). All-regions is the recommended default
- **Global service events** — **IAM, STS, CloudFront** (and **Route 53**) have no region, so their events deliver to **`us-east-1`**. The trail setting must be enabled (console default)
- **Destinations**: an **[S3](../services/s3.md) bucket** (compressed JSON, retained indefinitely, optional **SSE-KMS**) and/or **[CloudWatch Logs](../services/cloudwatch.md)** (unlocks search + metric filters + alarms)
- **Organizational trail** — created from the **management account** of an [Organization](../services/organizations.md); captures all member accounts centrally
- **Log file integrity validation** — SHA-256 hashes + hourly **signed digest files** prove logs weren't tampered with
- **Not real-time** — log files are delivered ~every **15 minutes**. For fast automated reactions, match **management events in EventBridge** rather than polling delivered files

> [!tip] Exam Tip
> Pattern-match: *"who made this API call / who deleted the bucket"* → **CloudTrail**. *"keep audit logs longer than 90 days / deliver to S3"* → **create a trail** (Event History isn't enough). *"audit S3 object-level access / Lambda invokes"* → **data events** (opt-in, extra cost). *"audit supported AWS API calls through VPC endpoints"* → **network activity events**. *"missing IAM / STS / CloudFront logs"* → **global service events → `us-east-1`**. *"audit every account in the org centrally"* → **organizational trail from the management account**. *"prove logs weren't altered"* → **log file integrity validation**. *"react in real time to an API call"* → **EventBridge on CloudTrail events**, not the ~15-min log files. *"detect unusual API activity automatically"* → **CloudTrail Insights**.

> [!warning] Common CloudTrail Gotchas
>
> - **Event History ≠ a trail.** It's free but capped at **90 days** and **management events only** — no data events, no S3 archive.
> - **Data and network activity events are off by default and cost extra** — management events are the free default.
> - **CloudTrail is not real-time** for log delivery (~15 min). Any "instant" requirement points to EventBridge, not delivered log files.
> - **Global service events go to `us-east-1`** — if IAM/STS/CloudFront logs are "missing," check that setting / that region.
> - **Don't confuse CloudTrail with CloudWatch Logs, VPC Flow Logs, or S3 access logs** — CloudTrail = AWS API/account audit, not application/OS/network logs.

## KMS

See [KMS](../services/kms.md) for full details.

- KMS is **regional + public-endpoint**; keys are scoped to a region and **never leave** the service (FIPS 140-2 Level 2)
- Direct `Encrypt` / `Decrypt` is capped at **4 KB** per call — for larger payloads use **`GenerateDataKey`** + envelope encryption
- `GenerateDataKey` returns **two copies of the same DEK**: a **plaintext** version (use immediately, then discard) and a **ciphertext** version (store alongside the encrypted data). KMS does **not** store DEKs
- For **symmetric** `Decrypt`, you do **not** pass `--key-id` — KMS reads it from metadata embedded in the ciphertext blob
- **Three key ownership tiers**: **AWS Owned** (invisible, no control), **AWS Managed** (visible like `aws/s3`, AWS controls policy & rotation), **Customer Managed** (full control — required if you need custom policy, rotation control, grants, or deletion)
- Rotation creates **new backing key material** but keeps the **same logical Key ID**; previous backing keys are retained so old ciphertexts stay decryptable
- **Aliases** (`alias/...`) decouple callers from the underlying Key UUID — repoint an alias to switch keys without touching code
- **Every KMS key has a key policy, and it is required.** Unlike S3, **IAM alone cannot grant access** — the key policy must explicitly trust the account root first
- Three access-control layers compose: **Key Policy + IAM Policies + Grants**. Grants are temporary, programmatic delegations used by AWS services (EBS attaching an encrypted volume, Auto Scaling launching from an encrypted AMI, …)
- **Cross-account KMS access**: Allow on the key policy (key's account) **and** Allow on the caller's IAM identity (calling account) — identical intersection rule to cross-account S3

> [!tip] Exam Tip
> Pattern-match: *"encrypt a 5 GB file with KMS"* → **`GenerateDataKey` + envelope encryption** (direct `Encrypt` would exceed 4 KB). *"need to edit the key policy / disable rotation / share with another account"* → **Customer Managed** key, not AWS Managed. *"S3 SSE-KMS / EBS encryption uses KMS — but how does KMS handle a 100 MB object?"* → the **service** does envelope encryption; KMS only wraps the DEK.

> [!warning] Gotcha
> Granting `kms:*` in an IAM policy is **not** enough — without a key policy statement trusting the account root, the call still fails. This is the opposite default from S3, where IAM permissions alone work within the same account.

> [!warning] Gotcha
> KMS Keys are **region-scoped**. Encrypting in `us-east-1` and trying to decrypt in `us-west-2` with the same Key ID fails — the key doesn't exist there. Use **multi-region keys** if cross-region decrypt is required.

### S3 Object Encryption — SSE & CSE

See [S3 — Object Encryption](../services/s3.md#object-encryption--sse-and-cse) for full details.

- **Buckets aren't encrypted — objects are.** "Bucket default encryption" only sets the default SSE flavour for new uploads; two objects in the same bucket can use different SSE methods
- **SSE is mandatory** on S3 (no opt-out since 2023) — the question is only *which* SSE method, not whether to encrypt
- **In transit (HTTPS)** is automatic and orthogonal to **at rest (SSE/CSE)** — both happen
- **CSE ≠ SSE-C.** SSE-C is still **server-side** — S3 does the crypto; you just provide the key per request. CSE means S3 never touches plaintext

| Method      | Key management                  | Encryption | Notable property                                          |
| ----------- | ------------------------------- | ---------- | --------------------------------------------------------- |
| **CSE**     | YOU                             | YOU        | S3 never sees plaintext                                   |
| **SSE-C**   | YOU                             | S3         | Key supplied per request; S3 stores only a one-way hash   |
| **SSE-S3**  | S3 (invisible)                  | S3         | AES-256; no key control; **no role separation**           |
| **SSE-KMS** | You + [KMS](../services/kms.md) | S3         | Rotation control + **role separation** + CloudTrail audit |

- **SSE-S3** = the default low-overhead option. S3 generates a per-object DEK, encrypts with **AES-256**, and wraps the DEK with an invisible internal master key. You give up key control, rotation control, and role separation
- **SSE-KMS** = uses [`GenerateDataKey`](../services/kms.md#data-encryption-keys-deks) against a Customer Managed KMS key. The encrypted DEK is stored in object metadata. This is the only SSE flavour that supports **role separation**, **rotation control**, and **CloudTrail-audited key usage**
- **SSE-C** = client supplies key on every PUT/GET. S3 keeps a hash to verify but destroys the key. Lose the key and the data is gone — by design

> [!tip] Exam Tip
> Pattern-match: *"AES-256"* → **SSE-S3**. *"Role separation / S3 admin must not be able to read object data"* → **SSE-KMS**. *"Audit who decrypts what / control rotation"* → **SSE-KMS**. *"Customer provides key per request, S3 stores only a hash"* → **SSE-C**. *"AWS must never see plaintext, ever"* → **CSE** (encrypt before upload).

> [!warning] Gotcha
> "Buckets are encrypted" is a trap framing. Encryption is **per-object**. If a question implies you can flip a bucket-level switch and have all existing objects re-encrypted, that's wrong — bucket default encryption only affects **new** objects from that point on.

> [!warning] Gotcha
> SSE-KMS uses [KMS DEKs](../services/kms.md#data-encryption-keys-deks) because KMS itself can only directly encrypt **≤ 4 KB**. Don't fall for *"how does KMS encrypt a 1 GB object?"* — it doesn't; S3 does the bulk encryption with a DEK that KMS generated.

#### S3 Bucket Keys — SSE-KMS cost optimization

See [S3 Bucket Keys](../services/s3.md#s3-bucket-keys-for-sse-kms-cost-and-throttling) for full details.

- **What it solves**: at scale (millions/billions of objects), SSE-KMS calls KMS once per PUT and once per GET — those request charges add up. **S3 Bucket Keys** can reduce KMS request charges by **up to 99%**
- **How it works**: S3 fetches a short-lived **bucket-level key** from KMS, caches it in memory, and uses it to derive **per-object DEKs locally** — no KMS round trip per object. The bucket key is still fetched **at least once per requester** so access is captured in CloudTrail
- **SSE-KMS only** — Bucket Keys don't apply to SSE-S3 (no KMS calls anyway), SSE-C (you supply the key), CSE (S3 doesn't encrypt), or DSSE-KMS (unsupported)
- **Only affects new objects** after enabling — to migrate existing objects, `CopyObject` re-encrypts them
- **Trust model unchanged**: bucket key is short-lived, never persisted, still derived from your KMS key

> [!warning] Gotcha — encryption context flips from object ARN to bucket ARN
> If you have a [KMS key policy](../services/kms.md#key-policies) or IAM policy with `kms:EncryptionContext:aws:s3:arn` conditions scoped to specific **object ARNs**, those conditions stop matching once Bucket Keys are on — S3 passes the **bucket ARN** instead. Update the policy conditions to use bucket ARN **before** enabling, or decrypts fail.

> [!warning] Gotcha — replication ETag drift
> Replicating an **unencrypted source object** into a destination bucket that has Bucket Keys + SSE-KMS as its default encryption re-encrypts the object on arrival — and the replica's **`ETag` differs from the source's**. Applications that compare source/replica ETags will break.

> [!tip] Exam Tip
> Pattern-match: *"SSE-KMS works but the KMS bill is huge on a bucket with millions of objects"* → **enable S3 Bucket Keys**. *"Cheaper than SSE-KMS"* on the same data is a misleading framing — Bucket Keys reduce **KMS request charges**, not the price per byte stored.

## SSM Parameter Store

See [Systems Manager — Parameter Store](../services/ssm.md#parameter-store) for full details.

- **Purpose** — central store for config and secrets; the **correct replacement for putting secrets in [EC2 user data](../services/ec2.md#bootstrapping)** (user data is readable by anyone who reaches the instance)
- **Types** — **String**, **StringList**, **SecureString** (SecureString = [KMS](../services/kms.md)-encrypted)
- **Hierarchical names** (`/app/db/password`) — retrieve **individually** by full name (`get-parameters`) or a **whole branch** at once (`get-parameters-by-path`); IAM permissions scope to a **single parameter or an entire tree**
- **Public AWS-published parameters** — classic use: latest **AMI ID** per OS/Region, referenced from [CloudFormation](../services/cloudformation.md) so templates don't hard-code an image
- **Regional + multi-AZ**, but **no cross-Region replication** — recreate parameters per Region
- **SecureString needs two permission layers** — `ssm:GetParameter*` **and** KMS permission on the key (`--with-decryption` triggers `kms:Decrypt`). Enables **role separation**: list-without-read. **Symmetric KMS keys only**
- **Standard vs advanced tier** — standard (4 KB, direct KMS `Encrypt`, **free**, 10K/Region) vs advanced (8 KB, **envelope encryption** via the Encryption SDK, **parameter policies**, **billable**, 100K/Region). Upgrade is **one-way** — no downgrade
- **Permission split** — standard encrypt `kms:Encrypt`; advanced encrypt `kms:GenerateDataKey`; decrypt (both) `kms:Decrypt`

> [!tip] Exam Tip
> Pattern-match: *"store config/secrets for EC2/Lambda, integrate with IAM, cheap, no rotation"* → **Parameter Store**. *"automatically rotate a DB password"* or *"native RDS secret integration"* → **[Secrets Manager](../services/secrets-manager.md)**. *"stop hard-coding the AMI ID in CloudFormation"* → **public parameter** (latest AMI ID). *"someone can list the secret name but must not read its value"* → SecureString with **KMS permission withheld** (role separation).

> [!warning] Gotcha
> Granting only `ssm:GetParameter` is **not enough** to read a SecureString — without `kms:Decrypt` on the key, the decrypt fails. The reverse trap: a parameter created in `us-east-1` is **not** visible in `us-west-2` (no cross-Region replication).

## AWS Secrets Manager

See [Secrets Manager](../services/secrets-manager.md) for the retrieval/rotation architecture and the application connection-pool example.

- **Purpose:** store and retrieve database passwords, API keys, and tokens without embedding them in code. [SSM Parameter Store](../services/ssm.md#parameter-store) also stores encrypted values; automatic credential rotation and service integration distinguish Secrets Manager.
- **Two credentials:** the application's [IAM role](../services/iam.md#iam-roles) authorizes secret retrieval; the returned database credentials authenticate its [RDS](../services/rds.md) connection.
- **Rotation updates both sides:** the secret value and the database/service credential must stay synchronized. Changing only the stored secret does not change the database password.
- **Two rotation mechanisms:** service-managed rotation for supported credentials such as RDS/[Aurora](../services/aurora.md) master passwords needs no rotation function; other cases use [Lambda](../services/lambda.md) templates/custom code. Lambda needs an execution role, network access, and appropriate target-system permissions.
- **Encrypted with [KMS](../services/kms.md):** a customer-managed key requires both Secrets Manager access and `kms:Decrypt` authorization, supporting separation of duties. KMS key rotation is separate from password rotation.
- **Application behavior:** rotation does not refresh a startup-loaded password automatically. Existing PostgreSQL sessions survive password changes; new physical connections need valid credentials.

> [!tip] Exam Tip
> Automatic database-password rotation points to Secrets Manager. Encrypted configuration without rotation can use Parameter Store. Neither “only Secrets Manager can store passwords” nor “all rotation requires Lambda” is correct.

## AWS WAF

See [AWS WAF](../services/waf.md) for the architecture diagram, rule actions, and the full pricing breakdown, and [Layer 7 Firewalls](../concepts/networking.md#layer-7-application-firewalls) for why NACLs and security groups cannot see HTTP content.

- **What it is:** AWS's managed **Layer 7** firewall for HTTP/HTTPS. The host service (CloudFront, ALB, API Gateway) terminates TLS and hands WAF the plain request, so WAF can read URI, headers, cookies, query strings, and the body.
- **Where it attaches:** CloudFront (and Amplify), with the Web ACL in **us-east-1**; or regional resources (ALB, API Gateway REST API, AppSync, Cognito user pool, App Runner, Verified Access) with the Web ACL in the **same Region**. Not an NLB, not EC2 directly. One Web ACL per resource; one Web ACL can serve many resources of the same scope.
- **Web ACL:** default action **Allow** (block known bad) or **Block** (allow only known good); rules evaluated in priority order. **1,500 WCU** included, up to **5,000** at extra cost (no support ticket). WAF WCU is rule complexity, not DynamoDB write capacity.
- **Rule groups:** AWS managed (mostly free), Marketplace (seller subscription), yours, or service-owned ([Shield Advanced](../services/shield.md), Firewall Manager). No default action; reusable across Web ACLs; capacity fixed at creation, max 5,000 WCU.
- **Rules = type + statement + action.** Regular or **rate-based** (per aggregation key, default source IP; 1/2/5/10-minute window; minimum limit 10; optional scope-down such as `/login`). Match on country, IP, label, headers, cookies, query, URI path, body, method; AND/OR/NOT.
- **Body inspection:** first **8 KB** on ALB and AppSync; **16 KB** default (up to 64 KB, extra cost) on CloudFront, API Gateway, Cognito, App Runner, Verified Access.
- **Actions:** Allow and Block **stop** evaluation; Count continues; CAPTCHA and Challenge continue for a valid token, otherwise block. **No Allow on rate-based rules.** Block can return a custom response; the others can add `x-amzn-waf-` request headers. **Labels** let a Count rule mark a request for a later rule in the same Web ACL.
- **Logs:** S3 (every 5 minutes), CloudWatch Logs, or Firehose; names must start with `aws-waf-logs-`. React fast with CloudWatch Logs or Firehose, not S3.
- **Pricing:** $5 per Web ACL per month, $1 per rule or rule group per month, $0.60 per million requests. Extras: Bot Control, CAPTCHA, Fraud Control (ATP, ACFP), Anti-DDoS, Marketplace groups, more than 1,500 WCU, larger body inspection.

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | SQL injection or XSS | WAF |
> | Rate limit per IP, HTTP flood, login brute force | WAF rate-based rule |
> | Block countries at Layer 7 | WAF geo match (or CloudFront geo restriction) |
> | Auto-update blocked IPs from logs or public lists | WAF plus Lambda/EventBridge feedback loop |
> | Same WAF rules in every account of an Organization | Firewall Manager |
> | Block an IP range for all protocols at the subnet | NACL |
> | Third-party firewall, IDS/IPS, non-HTTP inspection | Gateway Load Balancer |

## AWS Shield

See [AWS Shield](../services/shield.md) for the attack types, the full Standard versus Advanced comparison, and current pricing.

- **DDoS types:** L3 **volumetric** (saturate bandwidth, e.g. UDP reflection), L4 **protocol** (e.g. SYN flood: half-open connections block new ones), L7 **application** (cheap requests that are expensive to answer, e.g. HTTP floods on a search page).
- **Shield Standard:** free, automatic, always on. Common L3/L4 attacks at the Regional perimeter and the edge. Strongest with **Route 53, CloudFront, and Global Accelerator**. Does **not** stop HTTP floods: use WAF.
- **Shield Advanced:** **$3,000 per month per payer account** (once for an Organization's consolidated billing), **1-year commitment**, plus data transfer out fees.
- **Protects:** CloudFront, Route 53 hosted zones, Global Accelerator standard accelerators, ALB, CLB, and **Elastic IPs** (EC2 instances and NLBs only through their EIPs).
- **Not automatic:** each resource must be explicitly protected, or covered by a **Firewall Manager** Shield Advanced policy.
- **Cost protection:** credits for attack-driven scaling charges (EC2 Auto Scaling, CloudFront, Route 53, ALB, Global Accelerator); resource must be protected before the attack; request within 15 days after the billing month.
- **SRT:** 24/7 Shield Response Team with **Business or Enterprise Support**. **Proactive engagement** (SRT contacts you) needs **Route 53 health checks** on the resources and an emergency contact.
- **WAF integration:** L7 protection uses WAF; basic WAF fees for protected resources are included. Since March 2026 the **Anti-DDoS managed rule group** is the default for HTTP flood mitigation.
- **Also:** per-resource CloudWatch metrics (`DDoSDetected`), health-based detection (fewer false positives, faster detection), and **protection groups** (manage resources as one unit, new matching resources join automatically).

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Basic DDoS protection at no extra cost | Shield Standard (automatic) |
> | HTTP flood or login flood | WAF rate-based rule or Anti-DDoS managed rule group |
> | 24/7 DDoS experts, or AWS contacts us during an attack | Shield Advanced (SRT, proactive engagement) |
> | Credits for scaling costs caused by a DDoS attack | Shield Advanced cost protection |
> | DDoS protection across all accounts in an Organization | Firewall Manager Shield Advanced policy |

## AWS Config

See [AWS Config](../services/config.md) for the architecture diagram, rule types, remediation options, and pricing.

- **Two jobs:** record resource **configuration changes over time**, and evaluate resources against **Config rules** as compliant or non-compliant.
- **Detective, not preventative.** Config never blocks a change. To stop an action, use IAM or an [SCP](../services/organizations.md#service-control-policies-scps). Proactive evaluation returns a result before deployment, but the caller (a pipeline or CloudFormation hook) decides whether to deploy.
- **Configuration item (CI):** a resource's configuration at a point in time, with its relationships and the related CloudTrail event. All CIs for a resource are its **configuration history**. Kept 7 years by default; files delivered to an S3 bucket.
- **Config shows what changed; [CloudTrail](../services/cloudtrail.md) shows who made the call.**
- **Recording:** **continuous** (default, $0.003 per CI) or **periodic** (one CI per 24 hours if changed, $0.012 per CI). Periodic is cheaper only for resources that change more than four times a day. Record global IAM resources in one Region only.
- **Rules:** **AWS managed** (`s3-bucket-public-read-prohibited`, `encrypted-volumes`, `root-account-mfa-enabled`, `required-tags`) or **custom** with **Guard** or **Lambda**. Triggered by configuration changes, on a schedule, or both.
- **Integrations:** **SNS** for a configuration stream and compliance notifications; **EventBridge** receives compliance state changes (for example, to invoke Lambda).
- **Remediation:** native **SSM Automation documents**, manual or automatic (default 5 attempts). EventBridge to Lambda is a separate, non-Config option.
- **Regional.** An **aggregator** gives a read-only, multi-account, multi-Region view. **Conformance packs** deploy a set of rules and remediations together, including across an Organization.

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Track configuration changes of resources over time | AWS Config |
> | Who changed this resource? | CloudTrail |
> | Check resources against standards (public buckets, unencrypted volumes, required tags) | Config rules |
> | Automatically fix non-compliant resources | Config remediation with SSM Automation (or EventBridge to Lambda) |
> | Stop the non-compliant change from happening | IAM or SCP, not Config |
> | Compliance view across all accounts and Regions | Config aggregator |

## Amazon GuardDuty

See [Amazon GuardDuty](../services/guardduty.md) for the architecture diagram, protection plans, finding types, and tuning options.

- **What it is:** continuous **threat detection**. Combines **ML and anomaly detection** (learns your account's normal activity as a baseline) with **threat intelligence feeds** (known malicious IPs and domains). Identifies unexpected and unauthorized activity as **findings**.
- **Detective, not preventative.** GuardDuty never blocks traffic or changes resources.
- **Foundational sources:** [CloudTrail](../services/cloudtrail.md) management events, [VPC Flow Logs](../services/vpc-flow-logs.md), Route 53 Resolver DNS logs. GuardDuty reads its **own independent copy**: you do **not** enable Flow Logs or a trail for it, and it does not store the logs for you.
- **DNS findings** only when instances use the AWS-provided VPC resolver (the default). Your own DNS servers or a public resolver are invisible to GuardDuty.
- **Protection plans (optional):** **S3 Protection** (CloudTrail S3 data events; no trail data events needed), EKS audit logs, Runtime Monitoring (agent on EKS, ECS including Fargate, EC2), Malware Protection (EC2 EBS volumes agentless, S3 new uploads, AWS Backup), RDS Protection, Lambda Protection, AI Protection.
- **Response:** findings go to [EventBridge](../services/eventbridge.md), then **SNS** to notify or **Lambda** to remediate (for example, a NACL deny rule for the attacker's IP). New findings arrive in near real time; updates to existing findings every 6 hours by default (1 hour or 15 minutes optional).
- **Findings kept 90 days;** export to S3 to keep longer.
- **Multi-account:** administrator and member accounts. Recommended: a **delegated administrator** through [Organizations](../services/organizations.md) with **auto-enable** for new accounts (up to 50,000 members); or invitations (up to 5,000).
- **Regional.** Enable in **all Regions**, including unused ones. Use an SCP to stop member accounts from disabling it.
- **Tuning:** **trusted lists** (no findings) and **threat lists** (always findings) of IP addresses and, since 2025, domains; **suppression rules** auto-archive expected findings.
- **Suspend** keeps findings and settings; **disable** deletes them.

> [!tip] Exam Tip
>
> | Requirement | Answer |
> | --- | --- |
> | Intelligent threat detection across accounts with minimal setup | GuardDuty |
> | Detect EC2 instances talking to known malicious IPs or mining crypto | GuardDuty |
> | Automatically respond to a finding | EventBridge rule to Lambda |
> | Notify the security team about findings | EventBridge rule to SNS |
> | Enable and manage GuardDuty for every account, including new ones | Delegated administrator in Organizations, auto-enable |
> | Detect suspicious S3 data access or mass deletions | GuardDuty S3 Protection |
> | Software vulnerabilities on EC2 or container images | Amazon Inspector, not GuardDuty |
> | Personal data stored in S3 | Amazon Macie, not GuardDuty |
> | Resource misconfiguration against a standard | AWS Config, not GuardDuty |

## Key Service Properties to Memorize

| Service                                             | Resilience level                                                                                                       |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [IAM](../services/iam.md)                           | Global                                                                                                                 |
| [VPC](../services/vpc.md)                           | Regional                                                                                                               |
| IGW                                                 | Regional                                                                                                               |
| Subnet                                              | AZ-specific                                                                                                            |
| NAT GW                                              | Zonal (need one per AZ for HA)                                                                                         |
| [EC2](../services/ec2.md) instance                  | AZ-specific                                                                                                            |
| [S3](../services/s3.md) bucket                      | Regional (globally unique name)                                                                                        |
| [KMS](../services/kms.md) key                       | Regional (single-region by default; multi-region keys are a separate feature)                                          |
| [SSM Parameter Store](../services/ssm.md) parameter | Regional (multi-AZ within Region; no cross-Region replication)                                                         |
| [CloudTrail](../services/cloudtrail.md) trail       | Regional (all-regions trail spans every region; global service events → `us-east-1`)                                   |
| [DMS](../services/dms.md)                           | Regional; DMS Standard replication instances are placed in VPC subnets, while DMS Serverless hides instance management |

## Common Exam Traps

- [NACLs](../services/vpc.md#network-acl-nacl) apply to **subnet boundary** traffic — intra-subnet traffic is not filtered
- [Security Groups](../services/vpc.md#security-groups-sg) attach to **ENIs**, not instances or subnets
- NAT GW needs an **Elastic IP** (assigned automatically on creation)
- 1 VPC → at most 1 IGW; 1 IGW → at most 1 VPC
- DHCP Options Sets cannot be edited — create a new one and associate it
- [IAM](../services/iam.md) and network security ([SG](../services/vpc.md#security-groups-sg)/[NACL](../services/vpc.md#network-acl-nacl)) are independent layers — both must pass

## Edge Delivery: CloudFront, Lambda@Edge, and Global Accelerator

- [CloudFront](../services/cloudfront.md): distribution = overall configuration; behavior = ordered path match plus origin, cache policy, viewer protocol, signed-access rules, and function associations. First match wins; default is fallback.
- TTL validity differs from cache residency. `s-maxage` controls shared caches; versioned filenames avoid repeated invalidations, their potential charges, and old browser-cache reuse. Frequent releases plus cost efficiency → prefer versioned filenames; [invalidation billing](../services/cloudfront.md#invalidation-costs) counts submitted paths, not affected objects. File naming differs from [S3 versioning](../services/s3.md#versioning-and-mfa-delete).
- [OAC](../services/cloudfront.md#oac-sign-requests-on-behalf-of-a-distribution) attaches to an origin; the bucket policy grants `cloudfront.amazonaws.com` access with `AWS:SourceArn` identifying the distribution. OAC is signing configuration, not an identity or permission grant. Prefer it for SSE-KMS and modern S3 Regions.
- Protect both boundaries: OAC (preferred) or legacy OAI for an S3 bucket origin; trusted key groups plus signed URLs/cookies for viewers. S3 website endpoints are custom origins without OAC/OAI.
- Viewer and origin TLS are independent. A CloudFront viewer certificate in [ACM](../services/acm.md) uses us-east-1; an ALB origin certificate uses the ALB Region.
- [Lambda@Edge](../services/lambda-edge.md): viewer request runs before cache lookup; origin request runs for origin-bound requests. A/B rewrites and device/country/origin selection require correct cache partitioning. Check the current event-specific quotas before implementation.
- [Global Accelerator](../services/global-accelerator.md): AWS-assigned static anycast IPs lead to listeners, regional endpoint groups, and registered endpoints. Configure application DNS to use the accelerator; TCP/UDP acceleration does not cache content.

> [!tip] Exam Tip
> Cache HTTP objects → CloudFront. Stable anycast entry and global TCP/UDP paths → Global Accelerator. Transform CDN requests/responses → Lambda@Edge. [Edge Delivery](../concepts/edge-delivery.md) connects these choices.

## Practice Resources and Study Plan

Which practice exams to use and in what order. Use these once most of the technical sections above are familiar.

### Official Resources on AWS Skill Builder

AWS gives the Official Practice Question Set and the standard Exam Prep course away for free. The Official Practice Exam and the enhanced Exam Prep course need a Skill Builder subscription ([AWS Certification prep](https://aws.amazon.com/certification/certification-prep/)).

| Resource                       | Cost         | What it is                                                                                                                                                                                      |
| ------------------------------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Official Practice Question Set | Free         | 20 questions written by AWS that show the style of the real exam. It is the closest free match to real exam questions.                                                                          |
| Exam Prep course (standard)    | Free         | A 2-hour digital course that goes through each domain and reviews sample questions. Pair it with the Question Set.                                                                              |
| Official Practice Exam         | Subscription | Same question style and rigor as the real exam. It reports pass or fail and a scaled score, which helps you judge readiness. It is not the real exam, so the score is a guide, not a guarantee. |
| Exam Prep course (enhanced)    | Subscription | 6–8 hours of material: videos, hands-on labs, extra practice questions, and access to the Official Practice Exam.                                                                               |

The source note says the Question Set is also offered in other languages, including Traditional Chinese. That was not reverified here.

The [AWS Training FAQ](https://aws.amazon.com/training/faqs/) lists the Skill Builder Individual subscription at **29 USD per month or 449 USD per year**. The Team subscription is 449 USD per seat per year, with a minimum of 5 seats.

> [!tip] Subscription strategy
> One month at 29 USD is enough if you only need the Official Practice Exam. Subscribe in the last week or two before the exam, take the practice exam, and then cancel. Individual subscriptions renew automatically unless you cancel, and a full refund is only possible within 3 calendar days of the first purchase ([AWS Training FAQ](https://aws.amazon.com/training/faqs/)). Cancel as soon as you are done so you are not charged for a second month.

### Paid Third-Party Practice Exams

| Provider                                    | Price (approx.)                                | Size                          | Strengths                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------- | ---------------------------------------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tutorials Dojo (Jon Bonso)                  | 14.99 USD list on the TD portal, often on sale | 401 questions                 | The most widely recommended set. Detailed explanations with reference links and cheat sheets. Timed, review, section-based (by domain), topic-based (for example EC2 or S3), and final-test modes, where the final test draws 65 random questions. One year of unlimited access ([Tutorials Dojo](https://portal.tutorialsdojo.com/product/aws-certified-solutions-architect-associate-practice-exams/)). |
| Stephane Maarek and Abhishek Singh (Udemy)  | Udemy sale price                               | 390 questions (6 tests of 65) | Written from scratch and a companion to Maarek's SAA video course. Short and good for finding gaps. The note says it was last updated in April 2026.                                                                                                                                                                                                                                                      |
| Digital Cloud Training / Neal Davis (Udemy) | Udemy sale price                               | Several full exams            | Over 125,000 students. Explanations link to their online "Training Notes".                                                                                                                                                                                                                                                                                                                                |

Udemy blocked automated reads, so the two Udemy rows come from the source note (reviewed 2026-11-17) and were not reverified here.

> [!note] Udemy pricing
> Udemy courses are on sale for roughly 10–20 USD most of the time. Don't pay the list price.

### Free Third-Party Question Banks

Vendors and community authors write these, and quality and accuracy vary. Use them for extra practice volume, not to decide whether you are ready.

| Provider                        | What you get                                                                                                                                                                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Training Camp                   | A 50-question full test with a 90-minute timer and an explanation for every option, plus a 25-question quick test ([Training Camp](https://trainingcamp.com/aws-solutions-architect-practice-test/))                                        |
| Crucial Exams                   | Configurable tests: choose domains, 5–100 questions, and an optional timer (from the source note; not reverified)                                                                                                                           |
| Open-source SAA-C03 study guide | A community-maintained, MIT-licensed GitHub Pages site with 276+ practice questions, updated April 2026, and not affiliated with AWS ([study guide](https://chathurangavkd.github.io/AWS-Certified-Solutions-Architect-Associate-SAA-C03/)) |
| Tutorials Dojo blog             | Free sample question sets and an SAA-C03 study path                                                                                                                                                                                         |

### Recommended Study Sequence

1. Finish most of a core course, such as Adrian Cantrill's SAA-C03 course, and work through the technical sections of this page.
2. Take the free Official Practice Question Set (20 questions) to see the real exam style.
3. Practice with Tutorials Dojo:
   - Use section-based and topic-based modes to work on weak domains first. Use the [Content Domains](#content-domains) table to find the wiki pages for each domain.
   - Switch to timed mode for the last few full exams.
   - Scoring 80% or more in timed mode again and again is a strong sign you are ready. This is a common rule of thumb from the note, not an AWS figure.
4. In the final week or two, subscribe to Skill Builder for one month and take the Official Practice Exam as the last check. Then cancel the subscription.

Review the explanations for questions you got right as well as the ones you got wrong. Plenty of correct answers are lucky guesses between two options. The [Common Exam Traps](#common-exam-traps) section collects the distractors that show up most.

### Avoid Exam Dumps

> [!warning] Gotcha
> Avoid "real exam dump" sites. Their answers are often wrong, and using leaked exam content breaks the AWS Certification Agreement that candidates accept, which can lead to revocation of the certification. The agreement text was not reviewed in this ingest. The same sites are also behind most of the unofficial "SAA-C04" claims mentioned in [Format and Scoring](#format-and-scoring).

## References

- [IAM](../services/iam.md)
- [VPC](../services/vpc.md)
- [EC2](../services/ec2.md)
- [ELB](../services/elb.md)
- [S3](../services/s3.md)
- [Networking Concepts](../concepts/networking.md)
- [Identity and Access Concepts](../concepts/identity-and-access.md)
- [Resource Policies vs Identity Policies](../concepts/resource-vs-identity-policies.md)
- [Route 53](../services/route53.md)
- [DNS Concepts](../concepts/dns.md)
- [TLS/HTTPS Concepts](../concepts/tls-https.md)
- [Organizations](../services/organizations.md)
- [Control Tower](../services/control-tower.md)
- [Systems Manager (SSM)](../services/ssm.md)
- [Cost Optimization](../patterns/cost-optimization.md)
- [KMS](../services/kms.md)
- [CloudWatch](../services/cloudwatch.md)
- [CloudTrail](../services/cloudtrail.md)
- [CloudFormation](../services/cloudformation.md)
- [DMS](../services/dms.md)
- [Boot Time to Service Time](../patterns/boot-time-optimization.md)
- [Source note: SAA-C03 Practice Exam Resources](../../raw/aws-cert-study-notes/aws-solution-architect/saa_c03_practice_exam_resources.md) (reviewed by the author 2026-11-17)
- [AWS Certified Solutions Architect – Associate certification page](https://aws.amazon.com/certification/certified-solutions-architect-associate/) (checked 2026-11-18)
- [AWS SAA-C03 exam guide](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html) (checked 2026-11-18)
- [SAA-C03 exam guide (PDF)](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.pdf)
- [AWS Certification prep](https://aws.amazon.com/certification/certification-prep/) (checked 2026-11-18)
- [AWS Training FAQ: Skill Builder subscriptions](https://aws.amazon.com/training/faqs/) (checked 2026-11-18)
- [AWS SAA-C03 Step 1: Official Practice Question Set](https://training.resources.awscloud.com/associate-certification-articles/step-1-get-to-know-the-exam-and-exam-style-questions)
- [Tutorials Dojo: SAA-C03 practice exams](https://portal.tutorialsdojo.com/product/aws-certified-solutions-architect-associate-practice-exams/) (checked 2026-11-18)
- [Stephane Maarek: SAA-C03 practice exams (Udemy)](https://www.udemy.com/course/practice-exams-aws-certified-solutions-architect-associate) (not reachable for verification)
- [Digital Cloud Training: SAA-C03 practice tests (Udemy)](https://www.udemy.com/aws-certified-solutions-architect-associate-practice-tests-k) (not reachable for verification)
- [Training Camp: free SAA-C03 practice test](https://trainingcamp.com/aws-solutions-architect-practice-test/) (checked 2026-11-18)
- [Crucial Exams: SAA-C03](https://www.crucialexams.com/exams/aws/certified-solutions-architect-associate/saa-c03/practice-tests-practice-questions)
- [Open-source SAA-C03 study guide](https://chathurangavkd.github.io/AWS-Certified-Solutions-Architect-Associate-SAA-C03/) (checked 2026-11-18)
