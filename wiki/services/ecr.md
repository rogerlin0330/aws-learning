---
title: Amazon ECR
type: service
tags: [ecr, containers, container-image, container-registry, ecs, eks, fargate, docker, iam, inspector, image-scanning, cloudwatch, cloudtrail, eventbridge, replication, cross-region, cross-account]
certs: [aws-saa]
updated: 2026-11-23
---

# Amazon ECR

Amazon Elastic Container Registry (ECR) is AWS's managed [container](../concepts/containers.md) image registry. It is the AWS-native equivalent of Docker Hub: a place to store, version, and distribute container images so that container platforms such as [ECS](ecs.md), [EKS](eks.md), [Fargate](ecs.md#fargate-launch-type), or plain Docker hosts can pull and run them.

ECR is the registry half of the [container workflow](../concepts/containers.md#container-registries): you build an image, push it to ECR, and your orchestrator pulls it onto hosts to run as containers. The orchestrator decides *how* containers run; ECR is only concerned with *storing and serving the images*.

## Structure and Hierarchy

ECR is organized top-down, much like a source-control host such as Git or GitHub:

```text
Registry (public or private)
└── Repository (many per registry — like a Git repo)
    └── Image (many per repository)
        └── Tag (many per image — unique within the repository)
```

| Level          | Mental model                             | Notes                                                                                             |
| -------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Registry**   | The whole hosting service for an account | Each account has a **private** registry and access to a **public** registry. Top-level container. |
| **Repository** | One Git-style repo                       | Many per registry; one repository typically holds all versions of one application image.          |
| **Image**      | One built container image                | Many per repository.                                                                              |
| **Tag**        | A human-friendly version label           | Many per image; a **tag must be unique within its repository** (e.g. `latest`, `v1.2.3`).         |

> [!warning] Gotcha
> A private registry is **regional**, not global. Each account gets one private registry **per region**, addressed as `<account-id>.dkr.ecr.<region>.amazonaws.com`. Repositories and images live inside a specific region's registry; pushing to `us-east-1` does not make the image appear in `eu-west-1` unless you configure [replication](#replication). The "one private registry per account" shorthand is fine for the exam, but the region is part of the identity. See [Resource Scope](../concepts/resource-scope.md) for the broader regional-resource framing.

## Public vs Private Registries

The registry type sets the default permission model — specifically, whether anonymous reads are allowed.

| Registry type | Read (pull)                                              | Write (push)         |
| ------------- | -------------------------------------------------------- | -------------------- |
| **Public**    | Anyone — read-only, no permissions or credentials needed | Permissions required |
| **Private**   | Permissions required                                     | Permissions required |

In plain terms:

1. A **public** registry lets anyone pull images, but pushing still requires permissions. This is the model for distributing open-source or shared base images (the public gallery lives at `public.ecr.aws`).
2. A **private** registry requires permissions for **every** operation, pull and push alike. This is the default for organization-internal images.

> [!tip] Exam Tip
> Match on the access wording. *"Anyone can pull / open read access / publish a public base image"* points to a **public** registry. *"Permissions required for all operations / internal images only"* points to a **private** registry. All access — public or private — is governed through [IAM](iam.md).

## Security and Access Control

All ECR permissions are controlled through [IAM](iam.md), so the same identity-and-access machinery used everywhere else in AWS applies to image push and pull. There is no separate ECR credential system.

This is why ECR fits cleanly with the AWS container orchestrators:

- On [ECS](ecs.md#task-definitions-and-task-roles), the **task execution role** is what grants the platform permission to pull a private image from ECR (distinct from the task role used by the application).
- On [EKS](eks.md), nodes pull images from ECR using their IAM permissions; ECR is listed among the [native EKS integrations](eks.md#aws-service-integrations).
- ECR supports both **identity-based** policies (on the IAM principal) and **repository policies** (a [resource policy](../concepts/resource-vs-identity-policies.md) on the repository itself), which is how cross-account pull access is granted.

> [!warning] Gotcha
> Pull failures from ECS or EKS are frequently a missing-permission problem, not a networking one. On ECS specifically, confirm the **task execution role** can pull from the repository before suspecting the image or the network path.

## Image Scanning

ECR can scan stored images for software vulnerabilities (CVEs). There are two scanning tiers, and the distinction is a common exam target:

| Tier         | Engine                                                                   | What it covers                                                                                                                                                                                                                                                                                               |
| ------------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Basic**    | ECR's built-in scanner, using AWS native technology and the CVE database | **Operating system packages only.** Scans run on push or manually, at most once per image every 24 hours ([AWS: basic scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-basic.html)).                                                                                          |
| **Enhanced** | [Amazon Inspector](inspector.md#ecr-and-lambda-scanning)                 | Scans the **operating system AND programming language packages**, on a **layer-by-layer** basis. Use **continuous scanning**, which rescans as new CVEs are published, or scan on push only ([AWS: enhanced scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html)). |

Turning on ECR scanning in Inspector switches the registry from basic to enhanced scanning. Continuous rescanning covers only images pushed or used within Inspector's **re-scan duration** (7 days by default for accounts activated on or after October 15, 2026), so old, unused images eventually stop being rescanned ([AWS: scanning ECR images with Inspector](https://docs.aws.amazon.com/inspector/latest/user/scanning-ecr.html)).

> [!tip] Exam Tip
> Only **Enhanced** scanning uses **Amazon Inspector** and adds OS plus software-package, layer-by-layer coverage. *"CVE / vulnerability scanning of container images, including language-package dependencies"* points to **Enhanced scanning (Inspector)**. *"Basic, default scanning"* is the lighter built-in option. Do not attribute Inspector to the Basic tier.

## Observability and Integrations

ECR plugs into the standard AWS operational and event services:

| Capability                 | Service                     | What you get                                                                                                                                      |
| -------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Metrics**                | [CloudWatch](cloudwatch.md) | Near real-time metrics for authentication, push, and pull operations.                                                                             |
| **API audit log**          | CloudTrail                  | A record of all ECR **API actions** (who pushed, who pulled, who changed a repository).                                                           |
| **Event-driven workflows** | EventBridge                 | ECR emits **events** when images change, so you can trigger automation (e.g. start a deployment or a scan-based gate when a new image is pushed). |
| **Replication**            | ECR (built-in)              | Copy images **cross-region and cross-account** automatically.                                                                                     |

> [!tip] Exam Tip — match the job to the service
>
> - Vulnerability scanning, OS + packages, layer-by-layer → **Enhanced scanning (Inspector)**
> - Metrics on pull / push / auth volume → **CloudWatch**
> - Audit trail of who called which ECR API → **CloudTrail**
> - Automatically trigger a workflow when an image is pushed or updated → **EventBridge**
> - Get images into another region or another account → **Replication**

## Replication

ECR replication copies images automatically and works on **two independent axes**:

1. **Cross-region** — keep the same image available in multiple regions, for lower-latency pulls near the workload or for multi-region / DR deployments.
2. **Cross-account** — push once and make images available to other AWS accounts (for example, a central build account distributing to per-environment accounts).

Because a private registry is regional, replication is the mechanism that gets an image from the region where it was built to wherever it needs to be pulled. This is the registry-level parallel to [S3 Cross-Region Replication](s3.md#replication) for objects.

## ECR in the Container Picture

ECR is one piece of the AWS container story, and it is deliberately narrow:

- [Containers](../concepts/containers.md) — the underlying packaging model; ECR stores the **images** described there.
- [ECS](ecs.md) — AWS's proprietary orchestrator; task definitions point at images in ECR.
- [EKS](eks.md) — AWS-managed [Kubernetes](../concepts/kubernetes.md); pods run images pulled from ECR.
- [Fargate](ecs.md#fargate-launch-type) — the serverless compute layer under ECS/EKS still pulls its image from a registry such as ECR.

ECR does not build images, orchestrate containers, or run them. It stores them, secures them, scans them, and distributes them.

## Key Takeaways

1. ECR is AWS's managed container image registry — "Docker Hub for AWS."
2. Hierarchy: **registry → repository → image → tag**; tags are unique within a repository.
3. Each account has a **public** registry and a **private** registry, and the private registry is **per region**.
4. **Public** = anyone can pull, push needs permissions. **Private** = permissions needed for everything.
5. **IAM** controls all ECR access; repository (resource) policies enable cross-account pulls.
6. **Enhanced scanning uses Amazon Inspector** (OS + software packages, layer-by-layer); Basic is the lighter built-in scan.
7. Observability: **CloudWatch** (metrics), **CloudTrail** (API audit), **EventBridge** (image-change events).
8. **Replication** is both **cross-region** and **cross-account**.

## References

- Source: `raw/aws-cert-study-notes/aws-solution-architect/ecr_101.md`
- [AWS: ECR basic scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-basic.html) (checked 2026-11-23)
- [AWS: ECR enhanced scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html) (checked 2026-11-23)
- [AWS: scanning ECR images with Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/scanning-ecr.html) (checked 2026-11-23)
- [What is Amazon ECR? — AWS docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [Amazon ECR private registry — AWS docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html)
- [Image scanning in Amazon ECR — AWS docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)
- [Amazon ECR private image replication — AWS docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html)
