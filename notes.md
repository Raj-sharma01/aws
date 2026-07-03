route table is associalted with vpc and subnets. every vpc have a main (default) route table. subnets inherit route table from vpc.
route tables are not associated with resources.
if you add an entry in the vpc route table it will be used by all subnets in the vpc irrespective of public or private. so if entry for 0.0.0.0/0 to IGW is added to VPC technicall all private subnets will be able to use it and behaive liek public subnets.
Every VPC starts with a main route table.

For example

Main Route Table

10.0.0.0/16
→ local

When you create a subnet

Subnet A

inherits

Main Route Table
