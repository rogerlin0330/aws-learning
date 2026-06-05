---
title: Amazon EKS
type: service
tags: [eks, kubernetes, k8s, containers, fargate, ec2, ecr, node-group, managed-node-group, fargate-profile, control-plane, etcd, persistent-volume, eni, cloud-agnostic]
certs: [aws-saa]
updated: 2026-06-05
---

# Amazon EKS

Amazon Elastic Kubernetes Service (EKS) is **AWS's managed implementation of [Kubernetes](../concepts/kubernetes.md)**. AWS takes standard Kubernetes and runs it as a managed service, taking over the operationally painful pieces (the control plane and etcd) while you keep — and run on — the exact same Kubernetes you would find anywhere else.

The critical framing: **it is the same Kubernetes**, just extended to integrate cleanly with AWS. Everything in the [Kubernetes concept page](../concepts/kubernetes.md) — pods, services, ingress, kubelet, the control-plane components — applies unchanged. EKS does not invent new primitives; it manages existing ones.

## Why EKS Instead of ECS

Kubernetes is **cloud-agnostic**, and that portability is the whole reason to choose EKS over AWS's proprietary orchestrator, [ECS](ecs.md). The two main drivers:

- You need containers but **don't want vendor lock-in**.
- You **already run Kubernetes** (on-prem or another cloud) and want a managed AWS equivalent without rewriting to a proprietary system.

> [!tip] Exam Tip
> Keyword trigger: "cloud-agnostic," "avoid lock-in," "open standard," or "existing Kubernetes workload" points to **EKS, not [ECS](ecs.md)**. If the scenario is happy with an AWS-only, lower-overhead orchestrator, ECS is usually the cheaper, simpler answer.

> [!warning] Gotcha
> EKS charges an **hourly fee per cluster** for the managed control plane (roughly $0.10/hour, about $73/month per cluster on standard support), on top of the worker-node compute. [ECS](ecs.md) has **no control-plane charge** — you pay only for the EC2 or Fargate compute. For small or cost-sensitive workloads with no Kubernetes requirement, that per-cluster fee is a real reason the exam steers you to ECS.

## Where EKS Can Run

EKS is not only an in-AWS product. The deployment modes:

| Mode | What it means |
| --- | --- |
| **AWS** | Normal mode — EKS runs as a managed product inside AWS. *Assume this unless told otherwise.* |
| **AWS Outposts** | EKS running on Outposts — conceptually a small slice of AWS hardware on-premises. |
| **EKS Anywhere** | Create and run EKS clusters on-premises or anywhere else. |
| **EKS Distro** | The EKS Kubernetes distribution released as **open source**. |

> For the exam and the rest of this page, "EKS" means the **normal in-AWS deployment** unless a question specifies otherwise.

## The Control Plane

- **Managed by AWS** — you do not operate it.
- **Scales based on load.**
- Runs **across multiple Availability Zones** (resilient by design).
- **etcd** (Kubernetes' [key/value store](../concepts/kubernetes.md#control-plane-components)) is **also AWS-managed and distributed across multiple AZs**.

> [!warning] Gotcha
> Both the **control plane *and* etcd** are AWS-managed and multi-AZ. Don't attribute multi-AZ resilience only to the control plane — etcd's distribution across AZs is what makes the cluster state durable.

### Cluster endpoint access

The Kubernetes API server is reached through a **cluster endpoint**, and you choose how it is exposed:

- **Public** (default) — the API server is reachable over the internet (still IAM-and-token authenticated).
- **Private** — the API server is reachable only from within the VPC.
- **Public and private** — administrators reach it publicly while in-cluster `kubelet` traffic stays on the private path.

This matters for the architecture below: worker-node `kubelet` traffic can flow either over the **injected ENIs** or the **public endpoint**, depending on this setting.

## AWS Service Integrations

EKS integrates with AWS the way a native product would:

| Need | AWS service |
| --- | --- |
| Container images | **[ECR](ecs.md#container-registries-and-ecr)** (Elastic Container Registry) |
| Load balancing (wherever Kubernetes needs it) | **Elastic Load Balancers** (via the AWS Load Balancer Controller acting as the [ingress controller](../concepts/kubernetes.md#services-ingress-and-jobs)) |
| Security / identity | **[IAM](iam.md)** |
| Networking | **[VPC](vpc.md)** |

## EKS Cluster = Control Plane + Nodes

An **EKS cluster** is the AWS-managed **control plane** plus the **worker nodes** that run your pods.

### Node options

| Node type | Compute | Who manages provisioning & lifecycle |
| --- | --- | --- |
| **Self-managed nodes** | [EC2](ec2.md) (in a group) | **You** — billed at normal EC2 pricing |
| **Managed node groups** | EC2 | **EKS** handles provisioning and lifecycle |
| **Fargate** | Serverless | **AWS** — no instance provisioning, type selection, scaling decisions, or cluster packing to optimize |

**Fargate specifics:** you define **Fargate profiles**, which tell EKS which pods should launch onto Fargate. This is conceptually the same as [ECS on Fargate](ecs.md#fargate-launch-type) — no instance type to choose and no scaling decisions to make for that capacity.

> [!warning] Gotcha — capability support varies by node type
> Before committing to a node type, **verify it supports what you need.** Capabilities such as **Windows pods, GPU, Inferentia, Bottlerocket, Outposts, and Local Zones** are not uniformly supported across self-managed nodes, managed node groups, and Fargate. This has bitten real projects — always check the current AWS capability matrix rather than assuming.

## Persistent Storage

Recall from [Kubernetes](../concepts/kubernetes.md#storage-and-state) that storage is **ephemeral by default**. For [Persistent Volumes](../concepts/kubernetes.md#storage-and-state), EKS can use these storage providers:

- **EBS**
- **EFS**
- **FSx**

## Architecture Walkthrough

![Amazon EKS 101 architecture: an AWS-managed VPC holds the multi-AZ control plane (Kubernetes logo), while a separate customer VPC holds the EC2 worker nodes across AZ lanes; the control plane reaches the nodes via ENIs injected into the customer VPC, kubelet traffic flows over those ENIs or a public endpoint, an admin laptop reaches the control plane through the public endpoint, and service consumers reach the workload through ingress from the customer VPC](../assets/eks-architecture-101.png)

The deployment conceptually spans **two VPCs**:

1. **AWS-managed VPC** — where the **EKS control plane** runs, labelled "Control Plane — Managed & Multi-AZ." This VPC is owned and operated by AWS; it is **not** in your account.
2. **Customer VPC** — your account's [VPC](vpc.md), holding the **EC2 worker nodes** spread across AZs. If you use EC2 worker nodes, they live here.

The two are connected in three ways:

- **ENIs injected into the customer VPC.** The control plane reaches the worker nodes through Elastic Network Interfaces that AWS injects into your VPC.
- **kube-api traffic.** The `kubelet` on each worker node connects to the control plane — either over the injected ENIs **or** over the public control-plane endpoint, depending on the [endpoint access setting](#cluster-endpoint-access).
- **Admin via the public endpoint.** An administrator reaches the control plane through the public endpoint.

Service consumption flows the other way: end users reach the workload through **[ingress](../concepts/kubernetes.md#services-ingress-and-jobs) configurations that start from the customer VPC** and serve the consumer through a load balancer.

> [!tip] Exam Tip
> Remember the split: **control plane in the AWS-managed VPC** (multi-AZ), **EC2 worker nodes in your customer VPC**, connected by **injected ENIs** and/or the **public endpoint**. Admins use the public endpoint; service traffic exits via ingress from the customer VPC.

## Key Takeaways

1. EKS is **managed Kubernetes** — the same Kubernetes as everywhere, cloud-agnostic. Pick it to avoid lock-in or to bring existing Kubernetes workloads.
2. The **control plane and etcd** are both AWS-managed and multi-AZ.
3. EKS charges an **hourly per-cluster control-plane fee** that [ECS](ecs.md) does not.
4. Deployment modes: AWS (default), Outposts, EKS Anywhere, EKS Distro (open source).
5. Integrations: ECR (images), ELB (load balancing / ingress), IAM (security), VPC (networking).
6. Node options: self-managed EC2, managed node groups (EC2), and Fargate (serverless, via Fargate profiles).
7. Special capability needs (Windows / GPU / Inferentia / Bottlerocket / Outposts / Local Zones) constrain node choice — check support first.
8. Persistent storage providers: EBS, EFS, FSx.
9. Architecture: control plane in an AWS-managed VPC; EC2 workers in your VPC; connectivity via injected ENIs or the public endpoint.

## References

- [Kubernetes](../concepts/kubernetes.md) — the cloud-agnostic concept EKS implements
- [ECS](ecs.md) — AWS's proprietary container orchestrator (the EKS alternative)
- [Containers](../concepts/containers.md) — container vs VM model, images, registries
- [VPC](vpc.md) · [IAM](iam.md) · [EC2](ec2.md) — the integration surfaces
- [Amazon EKS — AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Amazon EKS pricing](https://aws.amazon.com/eks/pricing/)
- Source: `raw/aws-cert-study-notes/aws-solution-architect/eks_101.md`
