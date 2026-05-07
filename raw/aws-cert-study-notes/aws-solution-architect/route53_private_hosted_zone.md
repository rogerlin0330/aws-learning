# Route 53 Private Hosted Zone

![[R53_private_hosted_zone_infographic.png]]

[00:00:00] **Introduction to Private Hosted Zones in Route 53**  
This segment introduces **private hosted zones** in AWS Route 53, highlighting that they operate similarly to public hosted zones but with restricted access. Unlike public zones, private hosted zones are **not publicly accessible** and are instead **associated with one or more Virtual Private Clouds (VPCs)** within AWS.

- **Private hosted zones** are designed for DNS management **within specific VPCs**.
- They provide DNS resolution **only to resources inside associated VPCs**.
- Association can be done via the AWS Console, CLI, or API.
- Associations across different AWS accounts are possible using CLI and API only.

[00:00:22] **Functionality and Use Cases of Private Hosted Zones**  
- Private hosted zones behave like public zones in terms of creating resource records.
- Records in private zones are **resolvable only inside associated VPCs**.
- Support for **Split View (Split Horizon) DNS** is possible, allowing **different DNS responses for the same domain name depending on query origin** (internal vs. external).

Use cases:  
- Running a **corporate intranet** on the same domain as the public website but providing **different content internally**.
- Restricting access to certain systems so they are only accessible via DNS within the internal environment.

[00:01:16] **Architecture Overview of Private Hosted Zones**  
- Private hosted zones allow DNS queries **only from associated VPCs**.
- Public DNS queries (e.g., for netflix.com) work normally from the internet.
- Private hosted zones remain **inaccessible from the public internet**.
- VPCs use the **Route 53 resolver** to resolve DNS queries for associated private zones.

Key architectural points:  
- Only VPCs associated with the private hosted zone can resolve its records.
- VPCs not associated with the zone or external users cannot access these records.
  
[00:02:31] **Security and Access Considerations**  
- Private hosted zones are ideal when DNS records contain **sensitive information**.
- To access these DNS records, services **must run inside an associated VPC**.
- Association is mandatory for DNS resolution of private hosted zone records.

[00:02:53] **Explanation of Split View (Split Horizon) DNS**  
- Split View DNS allows using the **same domain name for both public and private hosted zones**.
- Example scenario:  
  - A VPC running Amazon Workspaces uses a private hosted zone with business-critical DNS records.
  - The private hosted zone is associated with this VPC.
  - From inside the VPC, DNS queries resolve to private records (e.g., accounting systems).
  - From outside, the public hosted zone with the same domain name resolves to public records only.

[00:03:42] **Detailed Example of Split View DNS Architecture**  
- The private hosted zone holds all internal records.
- A **public hosted zone with the same domain name contains a subset of these records**.
- External DNS queries follow the typical path:  
  ISP resolver → DNS root servers → Top-Level Domain (TLD) servers → Route 53 name servers.
- Public hosted zone records are accessible externally.
- Private-only records remain **inaccessible from the public internet**.

[00:04:37] **Practical Implications and Design Considerations**  
- This architecture is common for organizations needing to expose some DNS data publicly while keeping other records internal.
- It requires familiarity with DNS, Route 53, and AWS networking concepts.
- Developers, architects, and engineers should understand how to **integrate and implement split view DNS** effectively within AWS.

[00:05:01] **Conclusion**  
- The video summarizes the theory of private hosted zones and split view DNS.
- Encourages viewers to apply this knowledge in AWS environments.
- Prepares users for advanced topics and practical implementations in subsequent lessons.

---

### Summary Table: Key Features of Private Hosted Zones vs Public Hosted Zones

| Feature                          | Private Hosted Zone                          | Public Hosted Zone                    |
|---------------------------------|---------------------------------------------|-------------------------------------|
| Accessibility                   | Only within associated VPCs                   | Accessible from the public internet |
| Association                    | Must be associated with one or more VPCs     | N/A                                 |
| DNS Resolution                 | Resolvable only inside associated VPCs       | Resolvable globally                  |
| Cross-account association      | Possible via CLI/API only                      | N/A                                 |
| Use case                      | Internal DNS for sensitive/internal services | Public DNS for websites/services    |
| Support for Split View DNS    | Yes, can coexist with public zones on same name | Yes, for public access               |

---

### Key Insights

- **Private hosted zones provide controlled DNS resolution confined to AWS VPCs, enhancing security for internal resources.**
- **Split View DNS allows organizations to manage DNS zones with the same domain name differently for internal and external users, supporting complex enterprise requirements.**
- **Association of private hosted zones with VPCs is mandatory for DNS resolution, and services must run inside these VPCs.**
- **Cross-account VPC association with private hosted zones is supported, but only via CLI and API, enabling multi-account AWS architectures.**

---

### Core Concepts

- **Private Hosted Zone:** A DNS namespace in AWS Route 53 accessible only within specified VPCs.
- **VPC Association:** Linking a private hosted zone to one or more VPCs to enable DNS resolution within those networks.
- **Route 53 Resolver:** AWS service that processes DNS queries within VPCs, enabling resolution of private hosted zone records.
- **Split View (Split Horizon) DNS:** DNS setup where the same domain name resolves differently depending on query origin (internal vs external).
- **Public Hosted Zone:** Standard DNS namespace accessible globally on the public internet.

---

### Frequently Asked Questions (FAQ)

**Q:** Can private hosted zones be accessed from the public internet?  
**A:** No, private hosted zones are only resolvable from associated VPCs.

**Q:** Is it possible to associate a private hosted zone with VPCs in different AWS accounts?  
**A:** Yes, but only via the AWS CLI or API, not the console.

**Q:** What is Split View DNS and why is it useful?  
**A:** Split View DNS allows using the same domain name for both public and private hosted zones, providing different DNS records internally and externally. This is useful for intranet versus public website scenarios.

**Q:** How do DNS queries resolve in private hosted zones?  
**A:** Queries from resources inside associated VPCs use the Route 53 resolver to access private hosted zone records.

**Q:** Can a private hosted zone contain records not present in the public hosted zone with the same domain name?  
**A:** Yes, private hosted zones can contain internal-only records that are inaccessible from the internet.

---

This detailed summary captures all critical information about **AWS Route 53 private hosted zones**, their architecture, access control, and the concept of **Split View DNS**, strictly based on the provided content.
