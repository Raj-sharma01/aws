### VPC & Route Tables Basics

* Route tables are associated with VPCs and subnets. **Route tables are not associated with resources.**
* Every VPC starts with a **Main (default) Route Table**. Subnets inherit this main route table by default.
* If you add an entry (like `0.0.0.0/0` to IGW) to the *main* route table, it will be used by all subnets in the VPC. Technically, all those subnets will be able to use it and behave like public subnets.
* If you want some subnets to have something different in their routing, you need to create a **Custom Route Table** and explicitly associate it with those subnets.

### Public vs. Private Subnets

* The primary difference between a public and private subnet is the Route Table:
* **Public subnet:** Has a route to the IGW.
* **Private subnet:** Does not have a route to the IGW.


* Resources in a public subnet *can* have public IPs (they can be dynamic/auto-assigned, or static Elastic IPs).
* Traffic Flow:
* **Public subnet:** Resource -> IGW -> Internet
* **Private subnet:** Resource -> NAT GW -> IGW -> Internet



### Internet Gateway (IGW) & NAT Gateway

* **IGW:** Is not inside any subnet in a VPC. It is attached to the VPC boundary (it is the bridge between the VPC and the internet).
* **NAT Gateway:** Enables outbound-initiated communication.
* The EC2 instance starts the connection.
* `EC2  -------> Internet`
* `Internet ------X------> EC2` (No one on the internet can start a connection to your EC2).
* IGW is like a door to connect to the internet. it can't help if you don't have a public ip. to connect to internet you need both access to igw and a public ip.
* NAT(Network Address Translation) gw map private ip of recources withit own public ip and connects to the internet using igw. It maintains a mapping table where it source ip(private) and destination ip for each request. if a response comes from the destination ip it redirects it to private ip of the resources and then drops the entry from the mapping table. if any one from the internet try to access or send request to nat. it check if it is a response for any request using the map table, if its not it drops the packet. thus maintainig a one way internet acccess. I real world a router has both a NAT gw and IGW.


* A NAT GW **must** be placed in a public subnet to connect to the IGW, as normally only public subnets have the route for the IGW in their route table.

### Security Groups (SG) vs. Route Tables

* **Security Group (Permission):** Attached to the resource. It tells if the message packet is even *allowed* to reach that IP.
* To reach the internet, the EC2 SG must allow it (better to allow to a specific IP instead of `0.0.0.0/0`).
* **Outbound SG:** "Am I allowed to send traffic with this protocol, port, and destination?"
* **Inbound SG:** "Am I allowed to accept traffic with this protocol, port, and source?"


* **Route Table (Direction):** Associated with the subnet. It tells the packet *where it should go* to reach a specific IP.
* It is really a next-hop lookup table. "If I want to reach this destination IP, where should I send the packet next?"
* *Example:*
* Destination `10.0.2.15` -> Next hop: `local`
* Destination `8.8.8.8` -> Next hop: `NAT Gateway`





---

### Packet Flow Visualization

**Logical Flow:**
```text
 Application creates packet
 |
 ▼
Outbound SG *"Am I allowed to send this?"*
 │
 ▼
Route Table *"How do I reach that destination?"*
 │
 ▼
Next hop (NAT, IGW, local, etc.)
``` 
**Private Subnet to Internet Flow:**
```text
 EC2
 │
 ✔ Allowed by SG
 │
 Route Table
 │
 NAT Gateway
 │
 IGW
 │
 Google
```


### Route Tables (Additional Notes)

* Every subnet can be associated with **exactly one** Route Table at a time.
* One Route Table **can** be associated with multiple subnets.
* Associating a custom Route Table with a subnet **replaces** the Main Route Table for that subnet. The subnet does **not** use both.
* Every Route Table automatically contains a **local** route for the VPC CIDR (e.g. `10.0.0.0/16 -> local`).
* **`local` does NOT mean same subnet.** It means the destination is somewhere **inside the same VPC** (could be the same subnet or another subnet).
* Route Tables only look at the **Destination IP**. They do **not** consider:

  * Source IP
  * Protocol (TCP/UDP)
  * Port
  * Domain name (DNS)
* If multiple routes match, AWS always chooses the **most specific route (Longest Prefix Match)**.
* If no route matches the destination IP, the packet is dropped.

---

### Public vs. Private Subnets (Additional Notes)

* A subnet is considered **public** only if:

  * It has a route to an **Internet Gateway**, **AND**
  * The resource itself has a **Public IP** (or Elastic IP) if it needs to communicate directly with the internet.
* Simply attaching an IGW to a VPC does **not** make every subnet public.
---

### Internet Gateway (IGW)

* IGW is a **Regional** resource attached to exactly one VPC.
* It is **not** inside any subnet.

---

### Security Groups (Additional Notes)

* Security Groups are **stateful**.
* If an outbound request is allowed, the corresponding inbound response is automatically allowed.
* Likewise, if an inbound request is allowed, the outbound response is automatically allowed.
* Security Groups only have **Allow** rules. There are no explicit **Deny** rules.
* A resource can have **multiple Security Groups** attached.
* The effective permission is the **union (OR)** of all attached Security Groups.
* Security Groups can reference another Security Group instead of an IP range.

  * Instead of:

    * Allow MySQL from `10.0.1.0/24`
  * Use:

    * Allow MySQL from `AppServer-SG`
  * This avoids hardcoding IP addresses and automatically works for new resources using that SG.

---

### Network ACL (NACL)

* NACLs are attached to **Subnets**, not resources.
* Every subnet can have only **one** associated NACL.
* One NACL can be associated with multiple subnets.
* NACLs are **stateless**.

  * Every packet is evaluated independently.
  * If inbound traffic is allowed, the outbound response must also be explicitly allowed.
* Unlike Security Groups, NACLs support both **Allow** and **Deny** rules.
* If either the NACL or the Security Group blocks the packet, communication fails.
* NACLs are commonly used for:

  * Blocking known malicious IPs for an entire subnet.
  * Applying subnet-wide security policies.
  * Adding an extra layer of security (Defense in Depth).

---

Whenever an application inside your VPC queries an external domain or one of your own domains, it first queries AmazonProvidedDNS. If the query is for a public domain (like google.com), the resolver goes to the public internet to find the answer. If the query is for one of your own Route 53 domains (like ://mycompany.com), AmazonProvidedDNS forwards the request to Route 53, which acts as the ultimate authority for that domain and returns the mapped IP address

### Packet Flow (Updated Mental Model)

#### Browser → ALB → EC2

```text
Browser
 │
 DNS Lookup
 │
 Internet
 │
 IGW
 │
 ALB
 │
 Route (inside VPC)
 │
 NACL
 │
 EC2 SG
 │
 EC2
```

---

#### EC2 → Internet

```text
Application creates packet
 │
 ▼
Outbound SG
"Am I allowed to send this?"
 │
 ▼
Route Table
"Where is the next hop?"
 │
 ▼
NAT Gateway (if private subnet)
 │
 ▼
Internet Gateway
 │
 ▼
Internet
```

Packet Flow (Updated Mental Model)

Browser → ALB → EC2

```text
Browser
 │
 ▼
DNS Lookup
 │
 ├── Browser asks Local DNS Resolver (ISP/OS)
 │
 ├── If your domain is hosted in Route 53
 │       ▼
 │    Route 53 returns ALB IP/Alias
 │
 └── (Otherwise another public DNS provider)
 │
 ▼
Internet
 │
 ▼
Internet Gateway
 │
 ▼
ALB Security Group
 │
 ▼
ALB
 │
 ▼
Route (inside VPC - local)
 │
 ▼
EC2 Subnet NACL
 │
 ▼
EC2 Security Group
 │
 ▼
EC2
```

Private EC2 → Internet
```text
Application
 │
 ▼
DNS Lookup
 │
 ▼
AmazonProvidedDNS
 │
 ▼
Public DNS (if needed)
 │
 ▼
Destination IP obtained
 │
 ▼
Outbound Security Group
 │
 ▼
Route Table
 │
 ▼
NAT Gateway (if private subnet)
 │
 ▼
Internet Gateway
 │
 ▼
Internet
```

EC2 → RDS (Private)

```text
Application
 │
 ▼
DNS Lookup
 │
 ▼
AmazonProvidedDNS
 │
 ▼
Returns RDS Private IP
 │
 ▼
Outbound Security Group
 │
 ▼
Route Table (local)
 │
 ▼
RDS Subnet NACL
 │
 ▼
RDS Security Group
 │
 ▼
RDS
```
