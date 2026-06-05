---
title: Kubernetes
type: concept
tags: [kubernetes, k8s, containers, orchestration, control-plane, kubelet, etcd, pod, service, ingress, persistent-volume, cloud-agnostic, eks]
certs: [aws-saa]
updated: 2026-06-05
---

# Kubernetes

Kubernetes (K8s) is an open-source **container orchestration** system. It automates the deployment, scaling, and management of [containerized applications](containers.md) so they run reliably, scale on demand, make efficient use of compute, and stay reachable by users or other systems.

The useful mental model: it is like running [containers](containers.md) directly, but with automation doing the scheduling and an intelligent control loop doing the thinking — placing workloads, replacing failures, and reconciling actual state toward desired state.

Kubernetes is **cloud-agnostic**. It runs on-premises and on every major cloud (AWS, Azure, GCP), which is the whole reason it matters for AWS architecture: it is the portability story. AWS's managed implementation is [EKS](../services/eks.md), but the Kubernetes vocabulary on this page is identical everywhere Kubernetes runs — EKS just manages the pieces AWS can take off your hands.

> [!tip] Exam Tip
> "Container orchestration" plus "open standard," "cloud-agnostic," or "avoid vendor lock-in" points to Kubernetes (and on AWS, to [EKS](../services/eks.md)). A proprietary AWS-only orchestrator points to [ECS](../services/ecs.md).

## Cluster Structure

A **cluster** is a highly available set of compute resources organized to work as one unit. At the top level it has two roles:

- **Control plane** — manages the cluster: scheduling, application management, scaling, and deployment. This is the "thinking" layer.
- **Nodes** — virtual or physical servers that act as **workers**. Nodes are what actually run your containers.

![Kubernetes cluster structure: a single shaded cluster boundary containing three worker-node hexagons (each with a server icon and container cubes) and a central control-plane node, with kubelet agents on the nodes communicating to the control plane over the Kubernetes API](../assets/k8s-cluster-structure.png)

Every node runs, at minimum, two pieces of software:

- A **container runtime** (`containerd`, or another runtime such as Docker) that handles container operations.
- **kubelet** — an agent that talks to the control plane using the **Kubernetes API**.

So the top-level flow is: the control plane orchestrates containerized applications, and those applications run on nodes. All control-plane and node communication flows over the Kubernetes API.

> [!warning] Gotcha
> A real cluster has **many nodes**. A single-node cluster exists, but it is a test-only configuration with no resilience — do not treat it as a production shape.

## Control Plane Components

Zooming in, the control plane is a set of cooperating components. The node at the bottom runs the worker-side software.

![Kubernetes cluster detail: the control plane boundary holding five component hexagons (cloud-controller-manager, kube-controller-manager, etcd, kube-scheduler, kube-apiserver) above a worker node that runs containers in pods plus kube-proxy, with the node scaling out to NODE 2 through NODE N](../assets/k8s-control-panel-and-node.png)

| Component | Formal name | Role |
| --- | --- | --- |
| **API** | `kube-apiserver` | The **front end** for the control plane. Everything talks to it to reach the control plane. Can be **scaled horizontally** for performance and HA. |
| **etcd** | `etcd` | A **highly available key/value store** — the cluster's main backing store for all cluster data. |
| **Scheduler** | `kube-scheduler` | Continuously looks for pods **with no assigned node**, then assigns one based on resource needs, deadlines, affinity/anti-affinity, data locality, and other constraints. |
| **Cloud Controller Manager** *(optional)* | `cloud-controller-manager` | Integrates Kubernetes with a cloud provider (AWS/Azure/GCP). **Optional** — a small on-prem cluster won't have one. |
| **Controller Manager** | `kube-controller-manager` | A **collection of controller processes** (see below). |

The **kube-controller-manager** bundles several controllers:

- **Node Controller** — monitors and responds to **node outages**.
- **Job Controller** — runs pods to execute **jobs** (one-off tasks).
- **Endpoint Controller** — populates endpoints, **linking services to pods**.
- **Service Account & Token Controllers** — handle account and API-token creation.

### Node software (on every node)

- **Container runtime** (`containerd` / Docker / other) — handles container operations.
- **kubelet** — the agent that talks to the control plane via the Kubernetes API.
- **kube-proxy** — a **network proxy that runs on every node**. It coordinates networking with the control plane, helps implement **services**, and configures the rules that let traffic reach pods from inside or outside the cluster.

> [!warning] Gotcha
> **kube-proxy is a node component, not a control-plane component** — even though it implements services. Likewise, **cloud-controller-manager is optional**; it is only present when integrating with a cloud platform. Mixing these up is a classic exam trap.

> [!tip] Exam Tip
> Mnemonic for the control-plane set: **A.E.S.C.C.** — **A**pi, **E**tcd, **S**cheduler, **C**ontroller-manager, **C**loud-controller-manager.

## Pods — the Unit You Actually Work With

A **pod** is the smallest unit of computing in Kubernetes. Containers run *inside* pods, so the operating slogan is **"think pods, not containers."** You manage pods; the pod manages the container(s) inside it.

- A pod can hold **multiple containers** that share storage and networking, but the dominant pattern is **one-container-one-pod** — it is cleaner.
- Use multiple containers in one pod **only** when they are tightly coupled, need close proximity, and genuinely depend on each other.

### Pods are non-permanent (critical concept)

Treat pods as **temporary**: created to do a job, then disposed of. A pod can be:

- **Deleted** when finished,
- **Evicted** for lack of resources, or
- **Lost** if the node it runs on fails.

Pods are **not** designed to be durable or highly available. Other objects — services especially — provide the permanence. This is the direct Kubernetes expression of the [containers](containers.md) rule that the per-container read/write layer is scratch space, and it lines up with the [horizontal scaling](../patterns/scaling.md) principle that workloads should be stateless and replaceable.

> [!warning] Gotcha
> **Pods are NOT permanent or highly available.** Designing as if a specific pod will stick around is the single most common Kubernetes modeling mistake. The cluster replaces pods freely.

## Services, Ingress, and Jobs

Pods come and go, so you rarely point clients at a pod directly. Higher-level objects give you stable, consistent access:

| Term | Definition |
| --- | --- |
| **Service** | An **abstraction over pods** — this is usually what you think of as "the application." An app may be spread across many pods, but the service is the **consistent thing clients interact with.** |
| **Job** | An **ad-hoc** unit of work. Creates one or more pods, runs to completion, retries if needed, then finishes. Good for back-end / batch / isolated tasks — it is **not** a long-running service. |
| **Ingress** | How something **external** to the cluster reaches a service. |
| **Ingress Controller** | The software that arranges the underlying hardware to make ingress work. On AWS the **AWS Load Balancer Controller** provisions an ALB/NLB; NGINX is another common controller. |

The request path from the outside world to your application is:

```text
External users → Ingress → Service → Pod(s) → Container(s)
```

On AWS, the ingress controller is the seam where Kubernetes meets AWS networking — it turns an Ingress object into a real [Elastic Load Balancer](../services/ecs.md). See [EKS](../services/eks.md) for how this lands inside a customer [VPC](../services/vpc.md).

## Storage and State

> Architect pods to be **stateless**. Because pods are temporary, any long-running state needs somewhere durable to live.

- **State** can mean session data or data in the traditional sense.
- **Default Kubernetes storage is ephemeral**, provided locally by a node. If a pod **moves to another node, that storage is lost.** The AWS analogy is [EC2 instance store](../services/ec2.md#instance-store) (local, non-persistent) — *not* EBS.
- **Persistent Volumes (PVs)** provide storage whose lifecycle **outlives any single pod**. This is how you give containerized apps normal long-running storage. On AWS, [EKS](../services/eks.md) backs PVs with EBS, EFS, or FSx.

> [!warning] Gotcha
> Default pod storage is **ephemeral and node-local** — lost on pod relocation. Mentally map it to **EC2 instance store, not EBS**. Persistent storage requires a Persistent Volume.

## How Kubernetes Maps to AWS

Everything above is standard, cloud-agnostic Kubernetes. AWS's managed offering, [EKS](../services/eks.md), keeps the exact same vocabulary and takes over the operationally painful parts:

| Kubernetes concept | What EKS does with it |
| --- | --- |
| Control plane + etcd | AWS-managed and spread across multiple Availability Zones |
| Nodes | EC2 (self-managed or managed node groups) or serverless Fargate |
| Container images | Pulled from [ECR](../services/ecs.md#container-registries-and-ecr) (or another registry) |
| Ingress controller | AWS Load Balancer Controller provisioning ELB |
| Persistent Volumes | EBS, EFS, or FSx |
| Cloud Controller Manager | The integration point with AWS IAM, VPC, ELB |

## Exam-Oriented Keyword Triggers

| Phrase in the question | Answer |
| --- | --- |
| "Container orchestration" | **Kubernetes** (on AWS, [EKS](../services/eks.md)) |
| "Front end for the control plane" / "everything talks to it" | **kube-apiserver** |
| "Key/value store" / "backing store for the cluster" | **etcd** |
| "Assigns pods to nodes" / "pods with no node" | **kube-scheduler** |
| "Integrate with AWS/Azure/GCP" | **cloud-controller-manager** (optional) |
| "Responds to node outages" | **Node Controller** (inside kube-controller-manager) |
| "Network proxy on every node" / "implements services" | **kube-proxy** |
| "External access to a service" | **Ingress** (+ Ingress Controller for the hardware) |
| "Consistent abstraction over pods / the application" | **Service** |
| "Run to completion, then finish" | **Job** |
| "Persistent storage that outlives a pod" | **Persistent Volume (PV)** |

## Key Takeaways

1. Kubernetes is a cloud-agnostic container orchestrator; control plane does the thinking, nodes do the work.
2. Control-plane components: kube-apiserver (front door), etcd (state), kube-scheduler (placement), kube-controller-manager (controllers), cloud-controller-manager (optional cloud integration).
3. Node software: container runtime, kubelet, and kube-proxy — kube-proxy is a node component, not control-plane.
4. The pod is the smallest unit; think pods, not containers; one-container-one-pod is the common pattern.
5. Pods are non-permanent and not highly available — services provide the consistent abstraction.
6. Default storage is ephemeral and node-local (like instance store); Persistent Volumes outlive pods.
7. Request flow: external users → Ingress → Service → Pod(s) → Container(s).
8. AWS's managed implementation is [EKS](../services/eks.md) — same Kubernetes, AWS runs the control plane.

## References

- [Containers](containers.md) — container vs VM model, images, registries
- [EKS](../services/eks.md) — AWS's managed Kubernetes
- [ECS](../services/ecs.md) — AWS's proprietary container orchestrator (the EKS alternative)
- [Scaling — Horizontal vs Vertical](../patterns/scaling.md) — stateless, replaceable workloads
- Source: `raw/aws-cert-study-notes/aws-solution-architect/k8s_101.md`
