# EC2 Instance Types — Study Note

_AWS Certified Solutions Architect Associate_

---

## 1. Overview

An EC2 instance is an **operating system plus an allocation of resources**. The _instance type_ and _size_ give granular control over that allocation. Picking the right instance type matters — it can be the difference between good performance and a poor customer experience.

In the exam, instance-type questions often look like this: a performance problem is described and one of the answer choices is to change the instance type. The goal is to recognise which family fits which scenario and avoid clearly bad choices.

### What instance type selection influences

- **Raw resources** — virtual CPU, memory, local storage capacity, and storage type.
- **Ratios between resources** — compute-oriented instances give more CPU per dollar; memory-oriented instances give more RAM per dollar.
- **Cost per minute** of running the instance.
- **Network bandwidth** — both for data networking and for storage (important for EBS performance).
- **Underlying hardware architecture and vendor** — ARM vs x86, Intel vs AMD.
- **Additional capabilities** such as GPUs or FPGAs (Field Programmable Gate Arrays — programmable custom compute hardware).

---

## 2. The Five Instance Categories

EC2 instances are grouped into five categories. The category narrows down which family is appropriate for a workload.

- **General Purpose** — the default starting point. Even resource ratios. Use this unless there is a specific reason to move elsewhere.
- **Compute Optimized** — latest high-performance CPUs. More CPU than memory per dollar. Media encoding, HPC, scientific modelling, gaming, machine learning.
- **Memory Optimized** — inverse of compute optimised. Large memory per dollar. In-memory datasets, caching, certain database workloads.
- **Accelerated Computing** — extra hardware capabilities such as GPUs (parallel processing, ML) or FPGAs (programmable hardware). Niche — if needed, you will know.
- **Storage Optimized** — large amounts of fast local storage, optimised either for sequential throughput or for IOPS. Data warehousing, Elasticsearch, analytics.

---

## 3. Decoding an Instance Type Name

Instance type names look cryptic at first but follow a strict pattern. The whole string is the _instance type_ — use the full string when communicating to operations teams, because it is unambiguous.

![[ec2-instance-type-code-explanation.png]]

_Figure 1. Anatomy of an EC2 instance type name (R5dn.8xlarge)._

**Example: `R5dn.8xlarge`**

| Component                        | Example        | Meaning                                                                                                                                                              |
| -------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Family**                       | `R`            | Letter at the start. Identifies the family of instance (T, M, C, R, I, etc.). Here, R is short for "ram".                                                            |
| **Generation**                   | `5`            | Generation number. Higher = newer. Generally pick the newest available.                                                                                              |
| **Additional capabilities**      | `dn`           | Optional letters between generation and size. Common letters: `a` = AMD CPUs, `d` = NVMe local storage, `n` = network optimised, `e` = extra capacity (RAM/storage). |
| **Size**                         | `8xlarge`      | Determines the allocation of CPU and memory. Sizes range from nano, micro, small, medium, large, xlarge, 2xlarge, 4xlarge, 8xlarge, and upward.                      |
| **Instance type (whole string)** | `R5dn.8xlarge` | The full string is the instance type. Use the full string when communicating with operations teams — it is precise and unambiguous.                                  |

### Notes on each component

#### Family

A letter at the start identifies the family. Common ones: T (burstable), M (general purpose steady state), C (compute), R (memory / RAM), I (I/O), D (dense storage), G (GPU), P (parallel processing). Each family is designed for a specific type of workload.

#### Generation

AWS iterates often. A higher number is a newer generation, usually with better price-to-performance. As a rule, **pick the newest generation available**. Reasons not to: it is not available in your region, or internal test/approval processes still require the older generation.

#### Additional capabilities

Optional letters between the generation and the size signal extra capabilities. Worth committing to memory because exam time is limited:

- **`a`** — AMD CPUs
- **`d`** — NVMe local storage
- **`n`** — network optimized
- **`e`** — extra capacity (RAM or storage)
- **`g`** — Graviton (ARM) processors

#### Size

Within a family and generation there are multiple sizes, which scale CPU and memory roughly linearly. Sizes from smallest upward: nano, micro, small, medium, large, xlarge, 2xlarge, 4xlarge, 8xlarge, and beyond.

There is often a price premium at the largest sizes, so it is usually better to **scale horizontally** — many smaller instances rather than one huge one. This comes up again in the high availability and scaling lessons.

---

## 4. Reference Table of Instance Types

The diagram below is the quick reference to keep on hand throughout the course — print it out or keep it electronically. As each family is encountered later in the course, refer back to this and build the mental association between letters, categories, and use cases.

![[ec2-typical-instance-types.png]]

_Figure 2. Common EC2 instance types organised by category._

### Expanded reference

|Category|Common families|Use cases & notes|
|---|---|---|
|**General Purpose**|A1, M6g, T3, T3a, M5, M5a, M5n|Default starting point. Even CPU-to-memory ratio. A1/M6g use ARM (Graviton/Graviton 2) for cost-efficient workloads. T3/T3a are burstable — cheap for low average load with occasional peaks. M5 family is for steady-state workloads (M5a = AMD, M5n = network optimised).|
|**Compute Optimized**|C5, C5n|More CPU than memory per dollar. Media encoding, scientific modelling, high-performance computing, gaming servers, general machine learning.|
|**Memory Optimized**|R5, R5a, X1, X1e, High Memory (u-Xtb1), z1d|Large memory allocations per dollar. R5/R5a for real-time analytics, in-memory caches and in-memory database operations. X1/X1e offer the lowest $/GiB of memory in AWS. High Memory (u-Xtb1) provides the highest absolute memory of all AWS instances. z1d combines large memory and CPU with directly attached NVMe.|
|**Accelerated Computing**|P3, G4, F1, Inf1|Instances with extra hardware. P3 (Tesla v100 GPUs) for parallel processing and ML training. G4 (NVIDIA T4 Tensor) for ML inference and graphics-intensive work. F1 provides FPGAs for genomics, financial analysis and big data. Inf1 is custom-designed for ML inference (recommendation, forecasting, voice, conversation).|
|**Storage Optimized**|I3/I3en, D2, H1|Large amounts of fast local storage. I3/I3en use local NVMe SSD for NoSQL databases, warehousing and analytics. D2 uses dense HDD storage at the lowest price per disk throughput — good for data warehousing, Hadoop and distributed file systems. H1 provides high throughput with a balanced CPU/memory ratio for HDFS, MAPR-FS and Apache Kafka.|

---

## 5. Letter Mnemonics

Memorising every detail is not required for the exam. The minimum useful step is to associate each letter with its category.

|Letter|Mnemonic|
|---|---|
|**C**|**C**ompute|
|**R**|**R**AM (memory)|
|**I**|**I**/O|
|**D**|**D**ense storage|
|**G**|**G**PU|
|**P**|**P**arallel processing|
|**T**|**T**urbo / burstable (T3, T3a)|
|**M**|"**M**ain" — general purpose steady state|

---

## 6. Choosing an Instance Type — Decision Guide

- **Start in General Purpose.** Only move out of it if there is a specific reason.
- Workload needs maximum CPU per dollar → **Compute Optimized** (C5, C5n).
- Workload needs large memory or in-memory datasets → **Memory Optimized** (R5, X1, High Memory, z1d).
- Workload needs GPUs, FPGAs, or ML accelerators → **Accelerated Computing** (P3, G4, F1, Inf1).
- Workload needs lots of fast local storage or high IOPS → **Storage Optimized** (I3/I3en, D2, H1).
- Workload is mostly idle with occasional bursts → **T3 / T3a** burstable family.
- Workload is steady-state general purpose → **M5 / M5a / M5n**.
- Workload runs on ARM-compatible code and you want cost efficiency → **A1 or M6g** (Graviton).
- For EBS-attached workloads, check the instance has enough **storage network bandwidth** — the instance, not the volume, can be the limiting factor.

---

## 7. Useful Resources

- **Official AWS documentation** — full overview of every category, family, generation, and size: [https://aws.amazon.com/ec2/instance-types/](https://aws.amazon.com/ec2/instance-types/)
- **Third-party sortable reference** with pricing across operating systems and purchase options: [https://instances.vantage.sh/](https://instances.vantage.sh/) (formerly ec2instances.info). Sortable and filterable by attributes, EBS optimisation, OS compatibility, and more.

---

## 8. Key Exam Takeaways

- The whole string (e.g. `R5dn.8xlarge`) is the _instance type_. Use it in full when precision matters.
- Family letter + category is the most useful association to memorise: **C**=Compute, **R**=RAM/memory, **I**=IO, **D**=Dense storage, **G**=GPU, **P**=Parallel, **T**=burstable, **M**=general steady state.
- Prefer the latest generation unless a specific reason prevents it.
- Burstable (T3) is cheaper for nominally low workloads with occasional spikes. Use M5 for sustained steady-state load.
- For EBS-heavy workloads, check the instance has the storage bandwidth to match the volume performance.
- Scale horizontally with smaller instances when possible — large sizes carry a price premium.