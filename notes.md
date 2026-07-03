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
