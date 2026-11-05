---
tags:
  - aws
  - saa-c03
  - networking
  - vpc
  - monitoring
  - flow-logs
---

# VPC Flow Logs

## Overview

VPC Flow Logs record **metadata** about IP traffic flowing to and from network interfaces in a VPC — who talked to whom, on which ports/protocol, how much, and whether it was allowed.

- **Metadata only, never packet contents.** Think "anything observable from outside the packet": source/destination IP, ports, protocol, packet and byte counts, accept/reject.
- Need payload inspection? That's a packet sniffer on an instance (or 📚 VPC Traffic Mirroring — see [[VPC Traffic Mirroring]] if/when noted).
- **Not real-time.** There is always a delay between traffic occurring and the record appearing.

## Diagram Walkthrough

### Slide 1 — Architecture & capture points

![[vpc-flow-logs-architecture.png]]

- **Header / left column — capture hierarchy.** VPC → Subnet(s) → Interface(s), with arrows pointing down. Read this as the core rule: *a flow log captures from its attach point downwards*.
- **Centre — the VPC.** Two subnets:
  - **Blue (private) subnet** — two database instances (primary + standby), each with its own ENI. Green arrows from *both* ENIs converge on one flow-log icon at the subnet boundary → a **subnet-level** flow log covers every ENI in the subnet.
  - **Green (public) subnet** — the application instance, accessed by Bob (pink arrows = traffic path). A single green arrow from its one ENI to a flow-log icon → an **ENI-level** flow log covers only that interface.
- **Right — capture filter.** Flow logs can capture **ACCEPTED**, **REJECTED**, or **ALL** traffic.
- **Footer — destinations & records.** Both flow logs feed into S3 and CloudWatch Logs; each destination receives the same space-delimited flow log record. The fields are decoded on the next slide.
  - Note that `dstaddr` in the record is the instance's **private** IP, even though Bob reached it from the internet. For incoming traffic, `dstaddr` is always the ENI's private IPv4 address ([AWS — flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)).

### Slide 2 — Record fields, ICMP example, exclusions

![[vpc-flow-logs-records.png]]

- **Left — the default (version 2) record fields, in order:** `version` `account-id` `interface-id` `srcaddr` `dstaddr` `srcport` `dstport` `protocol` `packets` `bytes` `start` `end` `action` `log-status`. Pink = the most frequently used (addresses, ports, protocol, action).
- **Top right — the scenario.** Bob pings the application instance (ICMP).
- **Middle — two records, read left to right:**
  - Line 1: `srcaddr` = Bob (pink), `dstaddr` = instance (blue) → inbound request → **ACCEPT**.
  - Line 2: addresses **swapped** → the outbound reply → **REJECT**.
  - Ports are `0 0` because ICMP doesn't use ports. Protocol `1` = ICMP.
- **How to interpret the pair:** inbound accepted, reply rejected. A security group can't cause this — it's stateful, so an allowed request automatically allows the response. The reject must come from a **stateless NACL** that allows inbound ICMP but not the outbound reply.
- **Protocol numbers callout:** ICMP = **1**, TCP = **6**, UDP = **17**.
- **Bottom — not recorded:** traffic to/from `169.254.169.254` (instance metadata), `169.254.169.123` (Time Sync), DHCP, the Amazon DNS server, and Amazon Windows license activation. (Full current list under Course-vs-Reality.)

## Mechanics

### Attach points (scope)
| Attach at | Captures |
|---|---|
| VPC | Every ENI in every subnet of the VPC |
| Subnet | Every ENI in that subnet |
| ENI | That one interface only |

### Capture filter
`ACCEPT` · `REJECT` · `ALL` — chosen at creation.

### Destinations
- **S3** — direct access to log files; integrate with third-party tools or your own processing. Query in place with **Athena** (schema-on-read; you pay for data scanned + S3 storage) — suits a data team doing formal, ad hoc analysis.
- **CloudWatch Logs** — integrate with other AWS services, stream onward, view in the console or programmatically.
- **Amazon Data Firehose** — see ⚠️ below; not covered in the course.

### Reading ACCEPT / REJECT with SGs and NACLs
- `action` reflects whether **security groups or NACLs** permitted the traffic.
- **SG only** (stateful): request and response evaluated as one conversation → typically **one** record.
- **NACL** (stateless): request and response evaluated separately → can produce **two** records.
- 🎯 **ACCEPT followed by REJECT for the same flow (addresses swapped)** ⇒ a NACL is blocking the return direction.

## Course-vs-Reality

- ⚠️ **Destinations: three, not two.** Course says S3 and CloudWatch Logs only. AWS added **Amazon Data Firehose** (then Kinesis Data Firehose) as a direct destination, from which logs can go to OpenSearch, Redshift, or third-party tools ([AWS blog](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-amazon-vpc-flow-logs-kinesis-data-firehose/); [AWS — publish to Firehose](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-firehose.html)). If an exam answer lists only S3/CWL, it's still not wrong — but don't eliminate an option because it says Firehose.
- ⚠️ **How "not real-time" actually breaks down** (course gives no figures) ([AWS — flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)):
  - **Aggregation interval**: default max **10 min**, optionally **1 min**. Interfaces on **Nitro** instances always aggregate at ≤1 min regardless of setting.
  - **Delivery** after aggregation: typically ~**5 min** to CloudWatch Logs, ~**10 min** to S3 — best effort, can be longer.
- ⚠️ **Exclusion list is longer than the transcript's.** The transcript omits Amazon DNS (the slide includes it). Current AWS list also includes: mirrored *source* traffic, traffic to the VPC router's reserved IP, ARP, and traffic between an endpoint ENI and an NLB ENI ([AWS — flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-limitations.html)).
  - Nuance: only the **Amazon** DNS server is excluded — if you run your own DNS server, that traffic *is* logged.
- 📚 **Custom formats.** The 14 fields above are the fixed **default** format (version 2). A custom format can add later-version fields (e.g. `vpc-id`, `subnet-id`, `instance-id`, `tcp-flags`, `pkt-srcaddr`/`pkt-dstaddr` for traffic through NAT gateways or EKS pods) ([AWS — flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)).
- 📚 **Immutable after creation.** You can't change a flow log's configuration or record format — delete and recreate ([AWS — limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-limitations.html)).
- 📚 **Beyond VPC/subnet/ENI.** Flow logs can also be created for **Transit Gateways** and TGW attachments (documented separately in the Transit Gateway guide).

## Exam Traps

| 🎯 Trigger / scenario | Answer |
|---|---|
| "Inspect packet payload / contents" | **Not** flow logs — packet sniffer (or Traffic Mirroring 📚) |
| "Real-time network monitoring" | **Not** flow logs — delayed by design |
| Log all traffic across every subnet | Flow log at the **VPC** level |
| Monitor a single instance's interface | Flow log at the **ENI** level |
| ACCEPT then REJECT, same flow, addresses swapped | **NACL** blocking the return path (SGs are stateful) |
| SQL-style ad hoc queries over flow logs, pay per data scanned | **S3 + Athena** |
| Why no records for metadata-service / DHCP / Amazon DNS calls? | These are **excluded** by design |
| Protocol field values | ICMP **1**, TCP **6**, UDP **17** |

## Quick Reference

| Aspect | Detail |
|---|---|
| Captures | Metadata only (no payload) |
| Scope | VPC → Subnet → ENI (from attach point down) |
| Filter | ACCEPT / REJECT / ALL |
| Destinations | S3, CloudWatch Logs, Amazon Data Firehose ⚠️ |
| Latency | 1 or 10 min aggregation + ~5 min (CWL) / ~10 min (S3) delivery |
| Default record | 14 version-2 fields; custom formats add more |
| Key fields | srcaddr, dstaddr, srcport, dstport, protocol, action |
| Not logged | IMDS, Time Sync, DHCP, Amazon DNS, Windows licensing (+ more ⚠️) |
| Related | [[Security Groups]], [[Network ACLs]], [[Athena]], [[CloudWatch Logs]] |

## Open Questions (for Roger)

1. **Images:** I referenced the two slides as `vpc-flow-logs-architecture.png` and `vpc-flow-logs-records.png` — they arrived inline rather than as files, so drop them into your vault under those names (or tell me your preferred names).
2. **📚 additions:** keep the custom-format, immutability, and Transit Gateway bullets, or trim to course scope?
3. **Cross-links:** `[[Security Groups]]`, `[[Network ACLs]]`, `[[Athena]]`, `[[CloudWatch Logs]]`, `[[VPC Traffic Mirroring]]` — adjust to match your existing note titles.
