# Route 53 Public Hosted Zone

**Amazon Route 53** provides two types of DNS zones: **public hosted zones** and **private hosted zones**. This summary focuses on **public hosted zones**, which are DNS databases accessible from the public internet and AWS Virtual Private Clouds (VPCs).

## Core Concepts

- A **hosted zone** is a DNS database for a specific domain (e.g., *animalsforlife.org*), containing DNS records such as **A records**, **AAAA records**, **MX records**, **NS records**, and **TXT records**.
- Hosted zones are **authoritative** for their domains once delegated through **name server (NS) records** entered at the top-level domain (TLD) zone.
- Route 53 is a **globally resilient, distributed service**. Its DNS name servers are spread across multiple regions and share the same dataset, ensuring high availability even if some regions fail.
- Hosted zones can be:
  - Automatically created when registering a domain through Route 53.
  - Created manually if the domain is registered elsewhere but DNS hosting is desired through Route 53.
## Public Hosted Zones: Architecture and Functionality

![[R53_public_hosted_zone_infographic.png]]

- Upon creation of a **public hosted zone**, Route 53 allocates **four public name servers** for that zone.
- These name servers host the zone file and are reachable from:
  - The public internet.
  - AWS VPCs using the **Route 53 resolver** (assuming DNS resolution is enabled within the VPC). An Amazon VPC connects to a VPC Resolver at a VPC+2 IP address. This VPC+2 address connects to a VPC Resolver within an Availability Zone. A VPC Resolver automatically answers DNS queries for:
	- Local VPC domain names for EC2 instances (for example, ec2-192-0-2-44.compute-1.amazonaws.com).
	- Records in private hosted zones (for example, acme.example.com).
	- For public domain names, VPC Resolver performs recursive lookups against public name servers on the internet.
- In VPCs, DNS queries to the public hosted zone occur through the **VPC +2 resolver IP address**, enabling internal instances to resolve domain names hosted in Route 53.
- To integrate with the global DNS system, the domain's NS records at the registrar or TLD must be updated to point to these four Route 53 name servers.
- This setup allows any public internet user or authorized VPC instance to resolve DNS queries for the domain.
## DNS Resolution Process (Public Internet)

The DNS query resolution follows these steps:

| Step | Description                                                                                                    |
| ---- | -------------------------------------------------------------------------------------------------------------- |
| 1    | User’s resolver (e.g., ISP DNS server) queries the **root DNS servers**.                                       |
| 2    | Root servers provide delegation to the **TLD servers** (e.g., *.org* servers).                                 |
| 3    | TLD servers provide NS records for the domain (e.g., *animalsforlife.org*), pointing to Route 53 name servers. |
| 4    | The resolver queries the **Route 53 public name servers** for the hosted zone records.                         |
| 5    | The response is returned to the user’s resolver, completing the DNS lookup.                                    |

This hierarchical delegation is known as **"walking the DNS tree."**

![[R53_public_hosted_zones.png]]

## Cost Structure

| Cost Type       | Description                                                   |
| --------------- | ------------------------------------------------------------- |
| Hosted Zone Fee | Monthly charge for each hosted zone.                          |
| Query Fee       | Small additional fee for DNS queries against the hosted zone. |

- The charges are **minimal** for most use cases but can accumulate for high-volume sites.

## Additional Notes

- Public hosted zones can host domains **registered externally** (e.g., GoDaddy, Hover). The external registrar’s DNS settings must be updated to point to the Route 53 name servers allocated to the hosted zone.
- Hosted zones consist of **resource records** that contain the DNS data for the domain and subdomains (e.g., *www.animalsforlife.org*, MX records for email).

## Key Insights

- **Public hosted zones in Route 53 provide reliable, scalable DNS hosting accessible globally and within AWS VPCs.**
- **Delegation via NS records at the domain registrar or TLD is essential to make Route 53 authoritative for a domain.**
- Route 53’s distributed architecture ensures **high availability and resilience** even during regional outages.
- DNS resolution for public hosted zones follows the **standard DNS hierarchy**, starting from root servers down to Route 53 name servers.
- There is a **cost model based on hosted zones and queries**, which is generally economical but should be considered for large-scale deployments.
