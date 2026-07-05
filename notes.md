### VPC & Route Tables Basics

* Route tables are associated with VPCs and subnets. **Route tables are not associated with resources.**
* Every VPC starts with a **Main (default) Route Table**. Subnets inherit this main route table by default.
* If you add an entry (like `0.0.0.0/0` to IGW) to the *main* route table, it will be used by all subnets in the VPC. Technically, all those subnets will be able to use it and behave like public subnets.
* If you want some subnets to have something different in their routing, you need to create a **Custom Route Table** and explicitly associate it with those subnets.
* Associating a custom Route Table with a subnet **replaces** the Main Route Table for that subnet. The subnet does **not** use both.
* Every subnet can be associated with **exactly one** Route Table at a time. One Route Table **can** be associated with multiple subnets.
* Every Route Table automatically contains a **local** route for the VPC CIDR (e.g. `10.0.0.0/16 -> local`).
* **`local` does NOT mean same subnet.** It means the destination is somewhere **inside the same VPC** (could be the same subnet or another subnet).
* Route Tables (Direction) only look at the **Destination IP**. They are really a next-hop lookup table ("If I want to reach this destination IP, where should I send the packet next?"). They do **not** consider:
* Source IP
* Protocol (TCP/UDP)
* Port
* Domain name (DNS)


* If multiple routes match, AWS always chooses the **most specific route (Longest Prefix Match)**.
* If no route matches the destination IP, the packet is dropped.

---

### Public vs. Private Subnets

* The primary difference between a public and private subnet is the Route Table:
* **Public subnet:** Has a route to the IGW.
* **Private subnet:** Does not have a route to the IGW.


* A subnet is considered public **only** if it has a route to an Internet Gateway. However, for a resource inside it to actually reach the internet, the resource **must also** have a Public IP (or Elastic IP).
* Simply attaching an IGW to a VPC does **not** make every subnet public.
* Traffic Flow:
* **Public subnet:** Resource -> IGW -> Internet
* **Private subnet:** Resource -> NAT GW -> IGW -> Internet



---

### Internet Gateway (IGW) & NAT Gateway

* **IGW:** Is a **Regional** resource attached to exactly one VPC. It is not inside any subnet. It is attached to the VPC boundary (it is the bridge between the VPC and the internet).
* IGW is like a door to connect to the internet. It can't help if you don't have a public IP. To connect to the internet you need both access to IGW and a public IP.
* **NAT Gateway:** Enables outbound-initiated communication.
* The EC2 instance starts the connection.
* `EC2  -------> Internet`
* `Internet ------X------> EC2` (No one on the internet can start a connection to your EC2).


* NAT (Network Address Translation) GW maps the private IP of resources with its own public IP and connects to the internet using IGW. It maintains a mapping table where it stores source IP (private) and destination IP for each request.
* If a response comes from the destination IP, it redirects it to the private IP of the resources and then drops the entry from the mapping table. If anyone from the internet tries to access or send a request to NAT, it checks if it is a response for any request using the map table; if it's not, it drops the packet. Thus maintaining a one-way internet access. (In the real world a router has both a NAT GW and IGW).
* A NAT GW **must** be placed in a public subnet to connect to the IGW, as normally only public subnets have the route for the IGW in their route table.

---

### Security Groups (SG) vs. Network ACLs (NACL)

**Security Groups (Permission):**

* Attached to the **resource**. It tells if the message packet is even *allowed* to reach that IP.
* **Outbound SG:** "Am I allowed to send traffic with this protocol, port, and destination?"
* **Inbound SG:** "Am I allowed to accept traffic with this protocol, port, and source?"
* Security Groups are **stateful**.
* If an outbound request is allowed, the corresponding inbound response is automatically allowed (and vice versa).


* Security Groups only have **Allow** rules. There are no explicit **Deny** rules.
* A resource can have **multiple Security Groups** attached. The effective permission is the **union (OR)** of all attached Security Groups.
* Security Groups can reference another Security Group instead of an IP range (e.g., Use *Allow MySQL from `AppServer-SG*` instead of `10.0.1.0/24`). This avoids hardcoding IP addresses and automatically works for new resources using that SG.

**Network ACL (NACL):**

* NACLs are attached to **Subnets**, not resources.
* Every subnet can have only **one** associated NACL. One NACL can be associated with multiple subnets.
* NACLs are **stateless**.
* Every packet is evaluated independently.
* If inbound traffic is allowed, the outbound response must also be explicitly allowed.


* Unlike Security Groups, NACLs support both **Allow** and **Deny** rules.
* If either the NACL or the Security Group blocks the packet, communication fails.
* NACLs are commonly used for blocking known malicious IPs for an entire subnet, applying subnet-wide security policies, and adding an extra layer of security (Defense in Depth).

---

### DNS in VPC

* Whenever an application inside your VPC queries an external domain or one of your own domains, it first queries **AmazonProvidedDNS**. *(Exam Note: This resolver is always located at the base of the VPC IPv4 network range plus two. E.g., `10.0.0.2`)*
* If the query is for a public domain (like google.com), the resolver goes to the public internet to find the answer.
* If the query is for one of your own Route 53 domains (like mycompany.com), AmazonProvidedDNS forwards the request to Route 53, which acts as the ultimate authority for that domain and returns the mapped IP address.

---

### Packet Flow (Updated Mental Model)

#### Browser → ALB → EC2

```text
Browser
 │
 ▼
DNS Lookup
 │
 ├── Browser asks Local DNS Resolver (ISP/OS)
 │
 ├── If your domain is hosted in Route 53
 │        ▼
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
ALB Subnet NACL (Inbound)
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
EC2 Subnet NACL (Inbound)
 │
 ▼
EC2 Security Group
 │
 ▼
EC2

```

#### Private EC2 → Internet

```text
Application creates packet
 │
 ▼
DNS Lookup (AmazonProvidedDNS -> Public DNS)
 │
 ▼
Destination IP obtained
 │
 ▼
Outbound Security Group ("Am I allowed to send this?")
 │
 ▼
Route Table ("Where is the next hop?")
 │
 ▼
EC2 Subnet NACL (Outbound)
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

#### EC2 → RDS (Private)

```text
Application
 │
 ▼
DNS Lookup (AmazonProvidedDNS returns RDS Private IP)
 │
 ▼
Outbound Security Group
 │
 ▼
Route Table (local)
 │
 ▼
EC2 Subnet NACL (Outbound)
 │
 ▼
RDS Subnet NACL (Inbound)
 │
 ▼
RDS Security Group
 │
 ▼
RDS

```
