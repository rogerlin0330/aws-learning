# VPC

## VPC Core

A VPC is within 1 account & 1 region. There are two types of VPC, default and custom. An AWS account can only 1 default VPC, but can have multiple custom VPCs. Custom VPCs are 100% by default.

Default VPC

- CIDR is always 172.31.0.0/16
- /20 subnet in each AZ in the region
- With a default IGW and Network ACL attached

In general, using the default VPC is not a good practice for production environment, as:

- It it overlapping with the default Azure VPC
- It can overlaps with the subnets in your on-premises network.

### Design the Range of The VPC

Avoid common ranges - avoid future issues

- Using 10.x.y.z can be your preference, notice that you should
  - Avoid using 10.0.y.z, 10.1.y.z, 10.10.y.z
  - Avoid using 10.128.0.0 to 10.255.255.255 as 10.128.0.0/9 is the default range used by Google Cloud.
  - Somebody suggested that start with 10.16.y.z/16.
- Reserve 2+ networks per region being used per account. For example, assuming that we need 3 US, 1 Europe, and 1 Australia (total would be 5 regions), we want to have 2 IP ranges (i.e. 2 VPCs) in each region, and finally we need 4 AWS accounts. Thus, in total it would be: $5 \times 2 \times 4 = 40$ IP ranges. Thus, 10.16.0.0 to 10.127.255.255 should be safe to be used.

### Design the Size of the VPC

- How many tiers do you need? Usually, it will be web, app, db, + future (buffer)
- How many AZs do you need? Usually, each region has at least 3 AZs, and plus some buffer, lets say 1, and if your VPC range is /16, then you can use /20 for each subnet.
- Example setup:
  ![[vpc-subnet-tiers.png]]

### DNS in VPC

- Provided by Route53
- DNS server address is the VPC `Base IP + 2` , so if your VPC is 10.0.0.0/16, then the DNS server of your VPC is 10.0.0.2.
- Two important settings when creating VPC
  - `enableDnsHostnames` - true means it will give instances in the VPC a public DNS name if they have a public IP. The default is false for custom VPCs.
  - `enableDnsSupport` - true means enabling DNS resolution in VPC. The default is true for custom VPCs.

### Configurations in VPC

- DHCP Options Set
  - One option set applies to a VPC at one time
  - It is defined in the VPC, and flows through to its subnets.
  - It controls things like DNS servers, NTP servers, NetBIOS servers.
  - You can create DHCP Options Set, but you cannot edit it. To modify the DHCP Options Set of a VPC, you need to create a new one, and associate it to the VPC.

## Subnet

- Subnet is AZ resilient
- A subnet of a VPC - within a particular AZ. If that AZ fails, the entire subnet itself fails, so do any services that are only hosted in that subnet.
- 1 subnet ⇒ 1 AZ; 1 AZ ⇒ 0+ subnets
- The IPv4 CIDR of a subnet is a subset of the VPC CIDR
- The IPv4 CIDR of a subnet cannot overlap with other subnets in the same VPC
- Reserved IP addresses in every subnet (5 in total)
  - `Network address`. For example, 10.16.16.0 in subnet ‘10.16.16.0/20’.
  - `Network + 1 address`, i.e., VPC router address. For example, 10.16.16.1 in subnet ‘10.16.16.0/20’, as it is reserved as the VPC router, which is a logical network device that moves data between subnets, and in/out of the VPC if it is configured to allow that. The VPC router has a network interface in every subnet, and it uses this network + 1 address.
  - `Network + 2 address`, i.e., DNS nameserver address in the subnet. For example, 10.16.16.2 in subnet ‘10.16.16.0/20’.
  - `Network + 3 address`, reserved for future use
  - `Broadcast address`. For example, 10.16.31.255 (last IP in subnet)
- There is two options in subnet, which are:
  - Allow Assign Public IPv4
  - Allow Assign IPv6

## Route Table

TBA

## Internet Gateway

- VPC and IGW relationship: 1 VPC ← 0 to 1 IGW, 1 IGW → 0 to 1 VPC.
- IGW is regional resilient, meaning that you don’t need a IGW per availability zone.
- It gateways traffic between the VPC and the Internet or AWS Public Zone (S3..SQS..SNS..or other AWS public services)
- It is a gateway managed by AWS, i.e. AWS handles the performance
- If an instance inside a VPC is assigned for an IPv4 public IP address, that public address is not configured in the OS, but is configured into the IGW. However, for IPv6, since all of them are public addressable, so they are configured directly in the OS level of the EC2 instances.
  ![[vpc-igw-outbound.png]]
  ![[vpc-igw-inbound.png]]

## Stateful vs Stateless Firewalls

- Stateless firewall:
  ![[vpc-stateless-firewall.png]]
- Stateful firewall:
  ![[vpc-stateful-firewall.png]]
  - A STATEFUL firewall is intelligent enough to identify the REQUEST and RESPONSE components of a connection as being related.
  - ALLOWING the REQUEST (INBOUND or OUTBOUND), means **the RESPONSE (INBOUND or OUTBOUND) is automatically allowed**.

## Network Access Control List (NACL)

- A network ACL in AWS is associated with a subnet. Every subnet has an associated ACL.
- NACLs filter traffic crossing the subnet boundary (inbound and outbound)
- Connections within a subnet aren’t impacted by NACLs
- ACL has a list of inbound rules and outbound rules
- NACL is **stateless**! Thus, every communication (request & response) needs individual rules.
- Rules are processed in order, lowest rule number first. Once a match occurs, processing STOPS.
- NACLs can explicitly allow or deny traffics. * is an implicit DENY if nothing else matches.
  ![[vpc-nacl.png]]
- More complex situations. Thus, AWS prefers Security Group over NACL.
  ![[vpc-nacl-complex.png]]
- NACLs can filter traffics by IPs/CIDR, ports & protocols, but not by logical resources
- **NACLs cannot be assigned to AWS resources**, only subnets
- NACLs are usually use together with Security Groups to add **explicit DENY** (Bad IPs/Nets)
- Each subnet can have and only have one NACL (Default or Custom)
- A NACL can be associated with multiple subnets

## Security Groups

- Security group is stateful - detect response traffic automatically;
- Allowed (IN or OUT) request = allowed response
- **NO EXPLICIT DENY** ... only ALLOW or Implicit DENY
- can't be used to block specific bad actors (e.g. block a specific IP on port 443)
- Supports IP/CIDR ... and logical resources, including other security groups and itself
- Attached to ENI (Elastic Network Interface, a logical networking component in a VPC that represents a virtual network card), not instances nor subnets.
- SG topology:
  ![[vpc-security-group.png]]
- SG supports using logical references as source (inbound rules) or target (outbound rules). The benefits are listed below (screenshot 1). It also allows self reference (screenshot 2). 
  ![[vpc-security-group-logical-ref.png]]
  ![[vpc-security-group-logical-self-ref.png]]

## Network ACL (NACL) vs Security Group

| Feature                   | NACL                                                           | Security Group                                                                    |
| ------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Operates at**           | Subnet level                                                   | Resource/Instance level                                                           |
| **Stateful/Stateless**    | Stateless                                                      | Stateful                                                                          |
| **Rule Types**            | Allow AND Deny                                                 | Allow only (Implicit Deny)                                                        |
| **Rule Evaluation**       | Processed in order (lowest rule # first); stops at first match | All rules evaluated before decision; no ordering                                  |
| **Explicit Deny**         | Yes                                                            | No (only Implicit Deny)                                                           |
| **Return Traffic**        | Must be explicitly allowed (ephemeral ports!)                  | Automatically allowed                                                             |
| **Default (VPC-created)** | Allows all inbound & outbound                                  | Allows inbound from same SG; allows all outbound                                  |
| **Custom/New**            | Denies all inbound & outbound                                  | Denies all inbound; allows all outbound                                           |
| **Association**           | 1 subnet -> exactly 1 NACL; 1 NACL -> many subnets             | Up to 5 SGs per resource; many-to-many with resources                             |
| **Applies to**            | All instances in the associated subnet (automatic)             | Only instances explicitly assigned to the SG                                      |
| **Rule Scope**            | IPs/CIDR, ports, protocols                                     | IPs/CIDR, ports, protocols, **logical resources** (other SGs, prefix lists, self) |
| **Use Case**              | Broad subnet-level filtering; **block specific bad IPs**       | Fine-grained instance-level access control                                        |
| **Best Practice**         | Use with SGs to add explicit DENY for known bad actors         | Primary firewall for resource-level traffic control                               |

> [!tip] Exam Tip
> Since NACLs are **stateless**, you must allow both the request port (e.g. 443) AND the **ephemeral port range** (1024–65535) for return traffic. Security Groups handle this automatically because they are stateful.

## Network Address Translation (NAT) & NAT Gateway

- It is a set of processes for remapping SRC or DST IPs
- NAT is **IP masquerading** - hiding CIDR blocks behind one IP, i.e. give private CIDR range outgoing internet* access.
- Two types of NAT Gateway:
  - **Public NAT GW** (default) - lives in a **public subnet**, associated with an **Elastic IP**. Allows private subnet instances to reach the internet. Must route to an IGW.
  - **Private NAT GW** - lives in a private subnet, no EIP. Used for VPC-to-VPC or hybrid (overlapping CIDRs) communication via TGW/VGW.
- NAT GW is **stateful** - it tracks connections so it can map return traffic back to the original instance. Unsolicited inbound connections are dropped (no matching state entry).
- NAT GW is **many-to-one** (unlike IGW's 1:1 mapping) - multiplexes many connections behind a single IP.
- How a Public NAT GW works (outbound + return):
  - **Outbound (two translations)**:
     1. Instance sends: `(src: 10.16.32.10, dst: 1.3.3.7)`
     2. Private subnet route table has `0.0.0.0/0 -> nat-gw` -> VPC router forwards packet to NAT GW
     3. NAT GW rewrites src to its own **private IP** and records the mapping: `(src: NAT_GW_PRIV_IP, dst: 1.3.3.7)`
     4. NAT GW route table has `0.0.0.0/0 -> igw` -> packet reaches IGW
     5. IGW maps NAT GW's private IP to its **Elastic IP**: `(src: NAT_GW_EIP, dst: 1.3.3.7)` -> internet. Note: the EIP is associated with the NAT GW at the AWS resource level, but the NAT GW's ENI only carries its private IP. The IGW maintains the NAT_GW_PRIV_IP <-> NAT_GW_EIP mapping and performs the actual packet translation (same mechanism as EC2 public IPs).
  - **Return (reverse two translations)**:
     1. Response arrives at IGW: `(src: 1.3.3.7, dst: NAT_GW_EIP)`
     2. IGW maps EIP back to NAT GW's private IP: `(src: 1.3.3.7, dst: NAT_GW_PRIV_IP)`
     3. NAT GW looks up state table, rewrites dst back to original instance: `(src: 1.3.3.7, dst: 10.16.32.10)`
     4. Packet delivered to instance
- When creatign a NAT GW, an Elastic IP will be automatically be assigned to that NAT GW.
- Zonal NAT GW is AZ resilient, while Regional NAT GW is region resilient. If Zonal NAT GW is used and one wants region resilience, then each AZ needs to deploy a NAT GW.
- NAT GW is charged by duration & data volume. Partial hour will be charged as a full hour. 
- NAT GW vs NAT instance. EC2, by default, filters all traffic that it sends or receives. It drops any data packet that is on its network card when that network card is neither the source or the destination of that packet. Thus, to use a EC2 to work as a NAT instance, it will receive data packets that the source is the other resources in the VPC and the destination is the host in the Internet. Thus, it will neither be the source nor the destination. So, by default, that traffic will be dropped. To enable an EC2 instance to function as a NAT instance, one need to disable a feature called "source and destination checks". This can be disabled via the console UI, the CLI, or the API.
- In general, NAT GW is preferred over NAT instance in terms of availability, resilience, robustness. Only when cost is highly valued, a NAT instance is preferred over. A NAT instance can be significantly cheaper than NAT GW when the data volume is high.
- NAT GW does not support SG, NAT instance does. For NAT GW, only NACLs can be used to filter traffics.
    ![[vpc-nat-gw-vs-nat-instance.png]]
- NAT isn't required for IPv6 as they are all publicly routable in AWS. NAT gateways **don't work with IPv6**. In this case, use ::/0 Route + IGW for bi-directional connectivity.
- ::/0 Route + Egress-Only IGW (only works with IPv6) to give an IPv6 instance outgoing only access.

## AWS PrivateLink

- An endpoint (no matter what kind of endpoint, e.g., interface endpoint, resource endpoint, etc.) is an object within your current subnet that is directly connecting to other AWS service/resources that are outside of your current VPC or current subnet, but has a designated private IP in your current VPC.

## References

- [AWS VPC: Difference Between Internet Gateway and NAT Gateway](https://medium.com/awesome-cloud/aws-vpc-difference-between-internet-gateway-and-nat-gateway-c9177e710af6)
- [AWS — Amazon VPC Regional NAT Gateway (RNAT) Overview](https://medium.com/awesome-cloud/aws-amazon-vpc-regional-nat-gateway-rnat-overview-what-is-rnat-introduction-features-use-cases-benefits-fd978309c43a)
- [AWS Network ACL vs AWS Security Groups](https://claude.ai/share/1d7953df-f0ec-4e08-9357-c4b98510278a)
