VPC Endpoint
Services NOT inside your VPC (Public AWS Services)

These services belong to the AWS regional control plane or managed service plane.

Storage
✅ S3
✅ DynamoDB
✅ ECR (Registry)
✅ EFS Control Plane (the mount targets are inside your VPC)
Identity & Security
✅ IAM (Global)
✅ STS
✅ KMS
✅ Secrets Manager
✅ Systems Manager (SSM)
✅ Certificate Manager (ACM)
Monitoring
✅ CloudWatch
✅ CloudTrail
✅ EventBridge
✅ X-Ray
Serverless
✅ Lambda (the service itself)

Important

A Lambda function does not live inside your VPC.

If you configure Lambda with VPC access, AWS creates ENIs in your subnets so Lambda can reach private resources.

Think

Lambda Service
       |
       | creates ENI
       |
   Your VPC

Not

Your VPC
    |
 Lambda
Containers
✅ ECR Registry
✅ ECS Control Plane (control plane, not task)
✅ EKS Control Plane (unless using private endpoint, the managed control plane is still AWS-managed)

Your pods and worker nodes run in your VPC.
Messaging
✅ SNS
✅ SQS
✅ Amazon MQ
API Services
✅ API Gateway
✅ AppSync
Analytics
✅ Athena
✅ Glue
✅ EMR Serverless
AI Services
✅ Bedrock
✅ Rekognition
✅ Textract
✅ Comprehend
✅ Translate
✅ Transcribe


<img width="521" height="493" alt="image" src="https://github.com/user-attachments/assets/1c8336fc-f8b7-4d3b-99bd-03dd3fb76707" />

since these services live out side your VPC a common way to reach them is through IGW. While this traffic traverses the internet gateway, it does not leave the AWS network.

for resourcesin private subnet we can connect them to aws resources withoutany nat by using vpc endpoint.

| Gateway Endpoint                | Interface Endpoint (PrivateLink)      |
| ------------------------------- | ------------------------------------- |
| Free                            | Charged (hourly + data)               |
| Only S3 & DynamoDB              | Most AWS services + your own services |
| Added as a route in Route Table | Creates an ENI with a private IP      |
| No Security Groups              | Has Security Groups                   |
| Doesn't use DNS changes         | Usually accessed via DNS              |
| Doesn't create an ENI           | Creates an ENI in your subnet         |


- Gateway Endpoint
- Interface Endpoint (PrivateLink)
- Endpoint policies

VPC Peering
