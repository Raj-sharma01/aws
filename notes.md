route table is associalted with vpc and subnets. every vpc have a main (default) route table. subnets inherit route table from vpc.
route tables are not associated with resources.
Every VPC starts with a main route table. if you add an entry in the vpc route table it will be used by all subnets in the vpc irrespective of public or private. so if entry for 0.0.0.0/0 to IGW is added to VPC technicall all private subnets will be able to use it and behaive liek public subnets.

if you want some subnets to have something differnt in their route table then you need to create a custom route table for that subnet.
public subnet => resource -> IGW -> Internet
private subnet => resource -> NAT GW -> IGW -> Internet

public and private subnet have just two differences - 1. internet access through IGW and NAT (or no access) 2. resources in public subnet have (static) public IPs.

A NAT Gateway enables outbound-initiated communication.

The EC2 instance starts the connection.

EC2  -------> Internet

Internet ------X------> EC2

No one on the internet can start a connection to your EC2.

a NAT GW must be in a public subnet to connect to IGW as normally only public subnet have route for IGW in their route table.

IGW is not inside any subbnet in a VPC. it is attacthed to a VPC (bridge between VPC and internet)

