# Introduction to Containers

Container computing is a form of compute that sits alongside (and contrasts with) traditional VM-based virtualization. Understanding how containers work conceptually is the foundation for understanding AWS's container services.

---

## OS Virtualization and Why It's Wasteful

What we casually call "virtualization" is more precisely **operating system (OS) virtualization** — running multiple full operating systems on the same physical hardware via a hypervisor (e.g. an EC2 host running the Nitro hypervisor, with several virtual machines on top).

The key problem is that **each VM is a full OS plus its resources**, and the OS itself consumes a large share of those resources:

- On a VM with, say, 4 GB RAM and a 40 GB disk, the guest OS alone can consume **60–70% of the disk** and much of the memory.
- That leaves relatively little for the actual application and its runtime environment (libraries, dependencies).
- If many VMs run the **same or similar OS**, this is pure duplication and wasted capacity.
- Because the OS is heavy, **every operation** — start, stop, restart — has to manipulate the entire operating system.

The real goal is usually just to run applications in **separate, isolated, protected environments** — which does _not_ actually require a full, duplicated OS per application.

---

## How Containers Differ

Containers solve this with a different architecture:

|Aspect|VM Virtualization|Containers|
|---|---|---|
|Stack|Hardware → hypervisor → many guest OSs → apps|Hardware → host OS → container engine → containers|
|Isolation boundary|Full isolated OS per VM|Process within the host OS|
|OS per app|One full OS each|Shares the single host OS|
|Resource footprint|Heavy (OS dominates)|Light (just app + runtime)|
|Start/stop speed|Slow (whole OS)|Very fast|
|Density on one host|Lower|Much higher|

A **container engine** (e.g. **Docker**) runs on top of the host OS. A container is similar to a VM in that it provides an isolated environment for an application — but instead of running a whole separate OS on a hypervisor, **a container runs as a single process within the host OS**.

- It's isolated from other processes, but **uses the host OS** for heavy lifting like networking and file I/O.
- Inside, it behaves like an isolated OS: its own isolated file systems, and it can run isolated child processes (e.g. a web server or app server).
- It consumes very little memory/disk — only the **application plus its runtime elements** (libraries and dependencies).

Because you don't run a full OS per application, containers are far lighter than VMs, so **many more containers fit on the same hardware**. This **density** is one of the main benefits of containers.

---

## Image Anatomy

![[image-anatomy.png]]

A container is a **running copy of a Docker image** — analogous to how an EC2 instance is a running copy of its EBS volumes (its boot volume is booted into a running OS).

What makes Docker images special: they are **not a single monolithic disk image**. They're a **stack of multiple independent, read-only file system layers**.

### Building an image with a Dockerfile

Images are created from a **Dockerfile**. Each line is processed one by one, and **each line creates a new file system layer** containing only the differences it introduced (a **differential architecture**).

Using the example Dockerfile (a CentOS-based web server):

1. **`FROM centos:7`** — every image starts either **from scratch** or from a **base image**. Here the base is a minimal CentOS 7 distribution — a super-thin disk image with just enough to run an isolated copy of CentOS. → **Layer 1 (base).**
2. **`RUN yum ... install httpd`** — performs software updates and installs the Apache web server. → **Layer 2 (updates + web server).**
3. **`ADD` / `RUN` startup script** — adds a startup script with minor customizations. → **Layer 3 (customizations).**

The result is a Docker image made of three stacked, **read-only** file system layers.

> **Diagram orientation note:** In the slide, layers are drawn upside down for convenience. Strictly, the **base layer (red) belongs at the bottom**, updates + web server (blue) in the middle, and customizations (green) at the top.

---

## Container Anatomy

![[container-anatomy.png]]

A Docker image is **how you create a Docker container**. A container is a running copy of an image **with one crucial addition: an extra read-write (R/W) layer** on top.

- The image's file system layers are **read-only by default** and never change after creation.
- The added **R/W layer** is what allows the container to actually run. Everything that happens at runtime — log files, data the app generates or reads — is stored in this R/W layer.
- Each layer is **differential**: it stores only the changes versus the layers below. Stacked together, they form the single file system the container sees.

### Why this is efficient

Build a second container from the **same image** and it reuses the **identical read-only base layers** (CentOS 7, updates + web server, customizations). The containers are **identical apart from their R/W layer** — that R/W layer is the only thing that makes them separate and isolated.

- No duplication of the shared layers; being read-only, there's no risk of overwrites.
- Two containers from the same base might differ by only **a few MB** in their R/W layers, versus VMs whose separate disk images could be **tens or hundreds of GB**.
- This **scales**: 2 containers or 200, disk usage is minimized because the base layers are reused across all of them.

---

## Container Registries

Base layers (and complete images) are distributed through a **container registry** — a hub of container images. A popular public one is **Docker Hub**.

Typical workflow:

1. A developer/architect writes a **Dockerfile** and builds a **container image**.
2. They **push (upload)** the image to a repository — **private** (often requires authentication) or **public** (e.g. Docker Hub, generally open to the world). OS vendors publish their base images here too.
3. Images are **pulled (deployed)** to **Docker hosts** — servers running a container engine.
4. A Docker host can run **many containers** from one or more images, and a **single image can spawn containers on many hosts**.

One image → many containers, each unique thanks to its own solo R/W layer.

---

## Terminology Note: "Docker" vs "Container"

Because Docker is so popular, people often say **"Docker"** when they really mean **"container."** Strictly:

- A **Docker container** is _one type_ of container.
- A **Docker host** is _one type_ of container host.
- **Docker Hub** is _one type_ of container registry — operated by the company Docker.

There are other container engines, hosts, and registries; Docker is just the most common.

---

## Key Concepts (Refresher)

- **Dockerfiles** build **Docker images**; images are multi-layer file system images used to run **containers**.
- **Portability & consistency** — package an app with all its libraries into a container, and it runs **exactly as intended** on any compatible container host, with the same software versions.
- **Lightweight** — containers and images offload heavy lifting to the host OS, use very little memory, and start/stop very fast, while still providing much of the isolation of VMs.
- **Layer sharing** — image layers are **read-only**, an image is just a reusable collection of layers, and images can be based off other images. Large environments (hundreds/thousands of containers) can share a smaller set of base layers to save significant capacity.
- **Run only what's needed** — just the app and its dependencies, as a process in the host OS (no full OS).
- **Port exposure** — containers are isolated, so anything inside must be **explicitly exposed** to be reachable. A container can expose ports (e.g. **TCP 80** for HTTP) to make its services accessible from the host and the outside world.
- **Multi-container stacks** — complex applications can span **multiple containers** (e.g. a database container + an application container) to scale individual parts or implement multiple tiers.

---

## Exam Pointers

- A container runs as a **process on the host OS**, not as a full guest OS on a hypervisor — this is the core distinction from VM virtualization.
- Image layers are **read-only**; the **R/W layer is added at the container level**, and it's the only per-container difference when containers share an image.
- The **differential, layered architecture** is what drives container efficiency and density — expect questions framing this as a benefit over VMs.
- **Portability and consistency** are headline benefits to remember.
- Watch for loose use of **"Docker"** as a synonym for **"container"** in question wording.