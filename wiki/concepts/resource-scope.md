---
title: Resource Scope (Global / Regional / AZ / Edge)
type: concept
tags: [scope, regions, availability-zones, global, edge, dr, resilience]
certs: [aws-saa]
updated: 2026-06-09
---

# Resource Scope (Global / Regional / AZ / Edge)

Every AWS resource lives somewhere in a 4-tier hierarchy. Knowing the tier for a given service tells you three things at once: its **blast radius** when something fails, whether you need to **copy or replicate** it to use it elsewhere, and which **DR pattern** applies.

This page is the canonical scope reference. When a new service page is added to the wiki, append its scope here.

## Why scope matters

- **Blast radius.** An AZ failure takes down everything AZ-scoped in that AZ (EBS volumes, EC2 instances, subnets). A regional outage takes down the regional control plane (S3, KMS, the VPC itself). Global services are designed to survive a region loss; edge services tolerate POP failures transparently.
- **Movement cost.** Regional resources don't exist in other regions until you explicitly copy them (AMI copy, EBS snapshot copy, KMS multi-region replica, S3 CRR). AZ-scoped resources can't even cross AZs in the same region without a snapshot or AMI as the carrier — see [EC2](../services/ec2.md) for the snapshot/AMI relocation pattern.
- **ARN shape.** Global ARNs omit the region segment (`arn:aws:iam::123:role/Foo`). Regional ARNs include it (`arn:aws:s3:::bucket` is the exception — bucket names are global, but the bucket itself is regional).

## The four tiers

| Tier | What it means | Failure domain |
| --- | --- | --- |
| **Global** | Single namespace across all regions; no region segment in the ARN | Service-wide outage (rare) |
| **Regional** | Lives in exactly one region; multi-AZ resilient by default for most managed services | One region |
| **AZ-scoped** | Pinned to one Availability Zone; loss of that AZ loses the resource | One AZ |
| **Edge** | Delivered from a global network of Points of Presence (POPs); requests route to the nearest one | One POP (transparent failover) |

## Global resources

Single namespace, no region in the ARN. Control plane is multi-region by design.

| Resource / Service | Notes |
| --- | --- |
| [IAM](../services/iam.md) users, groups, roles, policies | All IAM principals and policies are global; one IAM database per account. STS endpoints exist per region but issue credentials valid globally |
| [Route 53](../services/route53.md) hosted zones, records | Authoritative DNS data is global; health checkers run from multiple regions |
| [Organizations](../services/organizations.md) | The org structure (root, OUs, SCPs) is global; member accounts are global entities |
| [Control Tower](../services/control-tower.md) | Governed from the management account across the org; the landing zone spans the regions you enable, and the **home region is always available** (cannot be denied) |
| [S3](../services/s3.md) bucket **namespace** | Bucket names are globally unique — but the bucket data itself is regional (see below) |
| CloudFront distributions | Global control plane; data is served from edge POPs (see Edge tier) |
| WAF (for CloudFront) | Global WebACLs attach to CloudFront; regional WebACLs attach to ALB/API Gateway |

## Regional resources

Live in exactly one region. Most managed services are multi-AZ resilient within the region. To use them in another region, you must **copy, replicate, or recreate**.

| Resource / Service | Notes |
| --- | --- |
| [VPC](../services/vpc.md) and its components (route tables, NACLs, IGW, security groups) | VPC is regional; subnets inside it are AZ-scoped (see below) |
| [EC2](../services/ec2.md) AMI | Region-specific; use **AMI Copy** to use in another region. New AMI ID is generated per region |
| [EBS](../services/ebs.md) snapshot | Stored in [S3](../services/s3.md) within the region; use **Snapshot Copy** to move cross-region |
| [S3](../services/s3.md) bucket (data) | Bucket lives in one region with multi-AZ resilience; CRR replicates to another region |
| [KMS](../services/kms.md) keys | Regional by default; **multi-region keys** are an opt-in feature that creates linked replicas in other regions |
| [CloudWatch](../services/cloudwatch.md) log groups, metrics, alarms | All CloudWatch data is per-region; cross-region dashboards aggregate, they don't replicate |
| [CloudTrail](../services/cloudtrail.md) trail | Regional service; an all-regions trail spans every region as one logical trail. **Global service events** (IAM, STS, CloudFront, Route 53) deliver to **`us-east-1`** |
| Lambda functions | Regional; cross-region invoke is allowed but the function itself doesn't exist elsewhere |
| DynamoDB tables | Regional; **Global Tables** is the opt-in cross-region replication feature |
| SNS topics, SQS queues | Regional |
| ELB (ALB / NLB / CLB) | Regional load balancer; targets can span multiple AZs in the region |
| Auto Scaling Group | Regional; can launch instances across multiple AZs within the region |
| EFS file system | Regional service (the file system itself), but **mount targets** are AZ-scoped (see below) |
| RDS DB instance | The DB resource is regional in terms of API, but the primary instance is placed in **one AZ** — Multi-AZ adds a standby in another AZ |

## AZ-scoped resources

Pinned to a single Availability Zone. Lose the AZ, lose the resource. To move across AZs in the same region, you typically need a snapshot or AMI as the carrier.

| Resource / Service | Notes |
| --- | --- |
| [EC2](../services/ec2.md) instance | AZ-bound through its host. Stop/start may land on a new host **in the same AZ**, never a different AZ |
| [EC2](../services/ec2.md) dedicated host | Physical AWS hardware in one AZ |
| [EBS](../services/ebs.md) volume | Lives in one AZ; can only attach to instances in that same AZ. Snapshot 1, restore to other AZ to "move" it |
| [EC2](../services/ec2.md) instance store | Physically attached to the host; not just AZ-scoped, but **host-scoped** — destroyed on stop or host migration |
| [VPC](../services/vpc.md) subnet | Each subnet is in exactly one AZ; CIDR cannot span AZs |
| [EC2](../services/ec2.md) ENI | Lives in one subnet, so one AZ; can be detached and reattached to a different instance in the same AZ |
| NAT Gateway (zonal mode) | Default mode: deployed into one subnet, dies with the AZ. Regional NAT Gateway is the multi-AZ alternative |
| EFS mount target | One per AZ; the file system is regional, but you need a mount target in each AZ you want to mount from |
| RDS DB instance (primary) | The actual running database process is in one AZ. Multi-AZ is a deployment option, not the default |

## Edge resources

Delivered from a global network of Points of Presence. You don't pick a region; AWS routes requests to the nearest POP.

| Resource / Service | Notes |
| --- | --- |
| CloudFront POPs | Cache and serve content from 400+ edge locations; origin is regional |
| Global Accelerator | Anycast IPs advertised from edge POPs; traffic enters AWS backbone at the nearest POP |
| Lambda@Edge | Functions deployed once via CloudFront, replicated to edge POPs |
| CloudFront Functions | Lighter-weight edge compute; runs at the POP itself |
| Route 53 resolution | DNS resolution served from edge POPs (the hosted zone data is global) |

## Common gotchas

> [!warning] Gotcha: S3 bucket scope is split
> The **bucket name** is global (globally unique namespace). The **bucket data** is regional (lives in one region with multi-AZ resilience). Both facts are true simultaneously — this is why you can't have two buckets with the same name in different regions, even though the data isn't replicated between them.

> [!warning] Gotcha: "Regional" managed services hide AZ-scoped components
> Many services advertised as regional are actually a regional control plane over AZ-scoped data planes. RDS, EFS, and even ELB nodes follow this pattern. "Regional" usually means "the API and metadata are regional," not "the workload survives any AZ failure automatically." Multi-AZ is often an opt-in deployment option, not the default.

> [!warning] Gotcha: AMI and snapshot are regional, not global
> An AMI created in `us-east-1` does not exist in `us-west-2` until you copy it. The copy gets a **new AMI ID** in the destination region. Same for EBS snapshots. This is a common DR oversight — backups in one region don't help if the whole region is down.

> [!warning] Gotcha: KMS keys are regional, encrypted data is portable
> A KMS key in `us-east-1` cannot decrypt anywhere else unless you use a **multi-region key**. But the encrypted ciphertext itself (e.g. an encrypted EBS snapshot) is just bytes — you can copy it cross-region, but you also need the key (or a multi-region replica) in the destination.

> [!warning] Gotcha: Stop/start stays in the AZ
> Stopping and starting an [EC2](../services/ec2.md) instance may move it to a different host, but **always within the same AZ**. There is no API to move an instance to a different AZ — you must snapshot the EBS root volume, create an AMI, and launch a new instance in the target AZ.

> [!tip] Exam Tip
> For SAA questions about disaster recovery, the first filter is usually scope: "the region is down" implies global services keep working, regional copies in another region keep working, and anything not pre-replicated is lost. "An AZ is down" implies anything AZ-scoped in that AZ is lost; anything Multi-AZ keeps working.

## References

- [AWS Regions, Availability Zones, and Local Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)
- [AWS service endpoints — regional vs global](https://docs.aws.amazon.com/general/latest/gr/aws-service-information.html)
- [Multi-Region application architecture](https://docs.aws.amazon.com/whitepapers/latest/aws-multi-region-fundamentals/aws-multi-region-fundamentals.html)
