### VPC & Route Tables Basics

* Route tables are associated with VPCs and subnets. **Route tables are not associated with resources.**
* Every VPC starts with a **Main (default) Route Table**. Subnets inherit this main route table by default.
* If you add an entry (like `0.0.0.0/0` to IGW) to the *main* route table, it will be used by all subnets in the VPC. Technically, all those subnets will be able to use it and behave like public subne[...]
* If you want some subnets to have something different in their routing, you need to create a **Custom Route Table** and explicitly associate it with those subnets.
* Associating a custom Route Table with a subnet **replaces** the Main Route Table for that subnet. The subnet does **not** use both.
* Every subnet can be associated with **exactly one** Route Table at a time. One Route Table **can** be associated with multiple subnets.
* Every Route Table automatically contains a **local** route for the VPC CIDR (e.g. `10.0.0.0/16 -> local`).
* **`local` does NOT mean same subnet.** It means the destination is somewhere **inside the same VPC** (could be the same subnet or another subnet).
* Route Tables (Direction) only look at the **Destination IP**. They are really a next-hop lookup table ("If I want to reach this destination IP, where should I send the packet next?"). They do **not*[...] 
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


* A subnet is considered public **only** if it has a route to an Internet Gateway. However, for a resource inside it to actually reach the internet, the resource **must also** have a Public IP (or Ela[...]
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


* NAT (Network Address Translation) GW maps the private IP of resources with its own public IP and connects to the internet using IGW. It maintains a mapping table where it stores source IP (private) [...]
* If a response comes from the destination IP, it redirects it to the private IP of the resources and then drops the entry from the mapping table. If anyone from the internet tries to access or send a[...]
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
* Security Groups can reference another Security Group instead of an IP range (e.g., Use *Allow MySQL from `AppServer-SG*` instead of `10.0.1.0/24`). This avoids hardcoding IP addresses and automatica[...]

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

* Whenever an application inside your VPC queries an external domain or one of your own domains, it first queries **AmazonProvidedDNS**. *(Exam Note: This resolver is always located at the base of the[...]
* If the query is for a public domain (like google.com), the resolver goes to the public internet to find the answer.
* If the query is for one of your own Route 53 domains (like mycompany.com), AmazonProvidedDNS forwards the request to Route 53, which acts as the ultimate authority for that domain and returns the ma[...]

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
<img width="738" height="559" alt="image" src="https://github.com/user-attachments/assets/ea3a8c70-4303-40a7-aabd-dc7acfbe5f21" />


### Elastic Network Interface (ENI)

#### Mental model
- An ENI is a virtual Network Interface Card (NIC) that provides an EC2 instance's network identity. Compute (CPU/RAM) and networking are separate: the EC2 supplies compute while the ENI supplies network identity.
- Every EC2 has at least one ENI (the Primary ENI). Some instance types support multiple ENIs.
- ENIs can be detached from one EC2 and attached to another within the *same Availability Zone (AZ)*, allowing network identity to move independently of compute.

#### What belongs to an ENI
The ENI (not the EC2) owns:
- Primary and Secondary Private IPv4 addresses
- MAC address
- Security Groups
- Any associated Elastic IP(s)

In short: everything related to networking belongs to the ENI.

#### Private IPs
- Each ENI has exactly one Primary Private IP and may have multiple Secondary Private IPs.
- An EC2 with multiple ENIs can therefore have private IPs on each ENI (each ENI keeps its own primary + optional secondaries).

Example:
- ENI-1: 10.0.1.15 (primary), 10.0.1.16 (secondary)
- ENI-2: 10.0.2.20 (primary), 10.0.2.21 (secondary)

#### Elastic IP (EIP) and Public IP mapping
- An Elastic IP is a static public IP allocated to your account and is associated with a Private IP on an ENI — not directly to the EC2.
- AWS performs a 1:1 NAT between the Elastic IP (public) and the Private IP (private).
- A Private IP can have at most one Elastic IP. If an ENI has multiple private IPs, different Elastic IPs may be associated with different private IPs on that ENI.
- Auto-assigned public IPs behave differently (transient on stop/start); Elastic IPs remain until detached or released.

Diagram (conceptual):
Internet ↔ Elastic IP (54.x.x.x) ↔ NAT ↔ Private IP (10.0.1.15) ↔ ENI ↔ EC2

#### Why move an ENI
- Detaching and reattaching an ENI preserves its networking characteristics (Private IP(s), MAC, Security Groups, associated Elastic IP(s)), so clients can keep connecting to the same private IP even if the compute (EC2) changes.
- Common uses: failover (move ENI to replacement instance), separating traffic, or connecting an instance to multiple subnets (within the same AZ).

#### Multiple ENIs and traffic separation
- Multiple ENIs allow:
  - Connections to different subnets (same AZ)
  - Logical separation of traffic (e.g., application vs management)
  - Additional private IP addresses across ENIs
- Limits on number of ENIs and IPs depend on the instance type.
### DHCP

**1. What is DHCP?**

* Know the basic idea: DHCP (Dynamic Host Configuration Protocol) automatically gives a device its network configuration when it joins a network. Instead of manually configuring IP address, Subnet mask, Default gateway, and DNS server, the device asks a DHCP server for them.

**2. Does AWS have a DHCP server?**

* Yes. Every VPC uses an AWS-managed DHCP service. When an EC2 starts, AWS automatically provides: Private IPv4 address, Subnet mask, Default gateway (VPC router), DNS server, and Domain name. You don't install or manage a DHCP server yourself.

**3. What is a DHCP Option Set?**

* This is the AWS-specific part. Normally AWS gives EC2 instances default values. A DHCP Option Set lets you customize some of those values (e.g., Domain name, DNS servers, NTP servers).
* It does **not** let you decide which private IP an EC2 gets, which subnet it joins, or which ENI it uses. (AWS DHCP doesn't assign specific IPs; assignment comes from the ENI/subnet).

**4. Does DHCP assign private IPs?**

* Technically: AWS allocates the private IP to the ENI. DHCP informs the operating system what IP it has.
* **Mental model:** `EC2 starts -> AWS allocates a private IP -> DHCP tells the OS: "Your IP, Your gateway, Your DNS server"`

---

## 5. EC2 Compute, Lifecycle & Storage

# EC2 Lifecycle

### Mental Model

An EC2 instance consists of two major parts:

* **Compute (temporary):** CPU, RAM, Physical Host, Instance Store (local SSD, if present)
* **Persistent Storage:** EBS Volumes (Root + Additional EBS)

When an instance is stopped, AWS can release the **compute** but keep the **EBS**.

### Instance States

* **Pending:** AWS is preparing the instance (allocating compute, attaching EBS volumes, configuring networking).
* **Running:** Instance is ready. Compute billing starts.
* **Stopping:** Instance is shutting down.
* Normally not billed.
* Exception: Hibernation is billed while RAM is being copied to EBS.


* **Stopped:** Compute is released. EBS volumes still exist.
* **Shutting-down:** Instance is being permanently deleted.
* **Terminated:** Instance no longer exists.

### Reboot

Think: **Restart the operating system.**

* Instance stays on the **same physical host**.
* CPU and RAM are restarted. RAM contents are lost.
* Root EBS, additional EBS, and Instance Store are preserved.
* Private IP and Public IP remain the same.

### Stop

Think: **Shut down the machine and give the hardware back to AWS.**

* Compute resources are released (CPU released, RAM lost, Physical host released).
* Root EBS and additional EBS are detached and preserved.
* **Instance Store is lost** because it belongs to the old physical host.
* Private IP remains the same. Auto-assigned Public IP changes when the instance starts again. Elastic IP remains associated.

### Hibernate

Think: **Stop + Save RAM.**

* Before stopping: `RAM -> Saved to Root EBS`
* When started again: `Root EBS -> RAM Restored -> Application continues`
* CPU/Physical host released. Root/Additional EBS preserved. Instance Store lost.
* Private IP remains the same. Auto-assigned Public IP changes. Elastic IP remains associated.
* No compute charges while Stopped, but you still pay for Root EBS, Additional EBS, and Storage used to save RAM.

### Terminate

Think: **Delete the instance permanently.**

* Compute is deleted. RAM/Instance Store is lost.
* Root EBS is deleted by default (configurable). Additional EBS depends on its Delete on Termination setting.
* Private and Public IPs are released. Elastic IP is disassociated (but not deleted).

### Storage Types

**EBS (Elastic Block Store)**

* Think: **Persistent network disk.**
* Network attached storage. Independent of the physical host. Can move with the instance to another host.
* Survives: Reboot, Stop, Hibernate. (Deleted on Termination by default).

**Instance Store**

* Think: **Local SSD of the physical server.**
* Physically attached to the host. Extremely fast. Temporary storage.
* Survives only: Reboot (same host).
* Lost on: Stop, Hibernate, Terminate.
* > **Rule:** Instance Store belongs to the **host**, not the **instance**.



---

## 6. EC2 Networking Identity (ENI & IPs)

### Elastic Network Interface (ENI)

#### Mental model

* Think of an ENI as a **virtual Network Interface Card (NIC)** that provides an EC2 instance's network identity. Compute (CPU/RAM) and networking are separate: the EC2 supplies compute while the ENI supplies network identity.
* Every EC2 has at least one ENI (the Primary ENI). Some instance types support multiple ENIs.
* ENIs can be detached from one EC2 and attached to another within the *same Availability Zone (AZ)*, allowing network identity to move independently of compute.

```text
EC2
│
├── CPU
├── RAM
├── EBS
│
├── ENI-1 (Primary)
└── ENI-2 (Optional)

```

#### What belongs to an ENI

The following networking properties belong to the ENI (not directly to the EC2):

* Primary and Secondary Private IPv4 addresses
* MAC address
* Security Groups
* Any associated Elastic IP(s)

> **Rule:** Everything related to networking belongs to the ENI.

#### Private IPs

* Each ENI has **exactly one Primary Private IP** and may have multiple Secondary Private IPs.
* An EC2 with multiple ENIs can therefore have private IPs on each ENI (each ENI keeps its own primary + optional secondaries).
* Example: `ENI-1: 10.0.1.15 (primary), 10.0.1.16 (secondary)` / `ENI-2: 10.0.2.20 (primary), 10.0.2.21 (secondary)`



#### Multiple ENIs & Traffic Separation

* Multiple ENIs allow connections to different subnets (same AZ), logical separation of traffic (e.g., application vs management), and additional private IP addresses across ENIs.

#### Why move an ENI?

* Detaching and reattaching an ENI preserves its networking characteristics (Private IP(s), MAC, Security Groups, associated Elastic IP(s)), so clients can keep connecting to the same private IP even if the compute (EC2) changes.
* Common uses: failover (move ENI to replacement instance).

### Public IP vs. Elastic IP

**Auto-assigned Public IP**

* Assigned automatically while the instance is running.
* Returned to AWS when the instance is stopped. New Public IP after Stop/Start.

**Elastic IP (EIP)**

* An Elastic IP is a **static public IP** allocated to your account and is associated with a Private IP on an ENI — not directly to the EC2.
* AWS performs a 1:1 NAT mapping between the Elastic IP (public) and the Private IP (private).
* A Private IP can have at most one Elastic IP. If an ENI has multiple private IPs, different Elastic IPs may be associated with different private IPs on that ENI.

```text
Internet ↔ Elastic IP (54.x.x.x) ↔ NAT ↔ Private IP (10.0.1.15) ↔ ENI ↔ EC2

```

> **Exam Shortcut:**
> * Keep the same **Private IP** → Think **ENI**
> * Keep the same **Public IP** → Think **Elastic IP**
> 
> 

---
### CORS (Cross-Origin Resource Sharing)

* Browsers enforce the Same-Origin Policy.
* If JavaScript on one origin (website/domain) tries to access another origin, the browser performs a CORS check.
* The server (e.g., S3) must return the appropriate CORS headers (such as `Access-Control-Allow-Origin`) to allow the browser to expose the response to JavaScript.
* CORS is enforced by the browser, not by server (S3) itself.
* Without CORS, S3 may successfully return the object, but the browser will block JavaScript from accessing the response.

### S3 - pre-signed URL
* Temporarily allow access (read or write) to a specific object without requiring AWS credentials.
* <img width="725" height="423" alt="image" src="https://github.com/user-attachments/assets/3790ea4c-3922-40d8-9ad1-2bbe11fc8649" />

