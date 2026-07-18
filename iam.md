### IAM Policies & Evaluation

* **Permission policy** is a JSON file defining what an identity can or cannot access. They use **implicit deny**.
* *(Note: When attached to an IAM Group, the policy defines the permissions for the users inside it, because a Group itself is not a principal).*


* In case of overlapping permission policies (allow and deny rules), **explicit deny always wins**. This is true for overlapping cases inside one policy or between multiple policies.
* *Example:* Let's say there is an IAM group "developer" with allow access to S3, and an "operations" group with an *explicit deny* access to S3. If you are in both groups, you won't be able to access S3.
* **Best Practice (Least Privilege):** Rely on implicit deny to manage regular access (only allow what is strictly needed). Only use explicit deny for strict security guardrails (e.g., explicitly denying access if a user doesn't have MFA enabled).


* There are two types of permission policies:
* **Identity-based:** attached to users, groups, and roles.
* **Resource-based:** attached directly to AWS resources (like S3 buckets).


* If an action is allowed by an identity-based policy, a resource-based policy, or both, then AWS allows the action. An explicit deny in either of these policies overrides the allow.
* **Conditions** are placed inside an optional "Condition" block at the end of a policy statement. They act as fine-grained filters using Condition Operators (like `StringEquals`, `IpAddress`) matched against Context Keys.

### Users & Groups

* Groups consist of users only, not any other group. Group-attached policies are inherited by users in that group.
* Groups are **not** a true identity. They can't be referenced as a principal in a policy. So we cannot give a resource a resource-based policy for an IAM group.
* A user may not belong to any group and have only direct or inline policies attached to them. A user can also have group policies (via inheritance) and direct policies.

### Accessing AWS

* How to access AWS:
1. **AWS Console:** protected via username + password + MFA (optional but recommended for non-root).
2. **AWS CLI:** protected via access key.
3. **AWS SDK:** protected via access key.


* Access keys are generated from the console (IAM), they are secrets like a password. Access keys have two parts:
* `access key id` => similar to username
* `secret access key` => similar to password


* All these methods belong to a user, and with these, you can only access what that user is allowed to access.

### AWS Roles & Temporary Access

* Roles are identities that are to be used by (mostly) AWS services, (different) AWS accounts, and federated users. Roles use temporary access.
* **Common Types of Roles:**
* **Service Roles:** Custom roles created to allow an AWS service (e.g., EC2, Lambda) to access other resources in your account.
* **Service-Linked Roles:** Predefined roles managed by AWS that are automatically created to grant a specific service the permissions it needs to function.
* **Cross-Account/Federated Roles:** Roles used to bridge access across distinct AWS accounts or external identity providers.


* You "assume" a role by making an API call to the STS service (`sts:AssumeRole`).
* The **IAM Evaluation Engine** checks if your current identity has a permission policy allowing `sts:AssumeRole` AND checks the target role's trust policy to see if you are allowed in.
* If IAM approves, STS returns an AccessKey, SecretKey, and SessionToken. **Using these temporary credentials is what it means to assume the role.**

<img width="718" height="345" alt="image" src="https://github.com/user-attachments/assets/cc0d2e23-e065-46af-9ef6-7542ee1ae648" />

<img width="1260" height="667" alt="image" src="https://github.com/user-attachments/assets/ff079b1c-3760-4a3b-8b20-d6472ab6cbef" />



### Trust Policies vs. Permission Policies

* IAM roles have two types of policies - **Trust Policy** (who can assume this role) and **Permission Policy** (what this role can do).
* **Trust Policy:** A required JSON policy attached to an IAM Role that explicitly defines which external entities (principals) are authorized to assume that role. It serves as a security gatekeeper ("Who is allowed to put on this hat?").
* In a trust policy (or any resource policy), you must mention a principal. You can specify: AWS account and root user, IAM roles, Role sessions, IAM users, Federated user principals, AWS services, or All principals.

trust policy example:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```

### How AWS Resources Assume Roles (No Infinite Loop)

* You do **not** need a role to assume a role when dealing with AWS infrastructure. AWS services have their own native identity.
* When creating a role for an AWS resource (like EC2), the principal in the Trust Policy is the AWS service itself (`ec2.amazonaws.com`). The service under the hood is natively allowed to call STS on your behalf.

### EC2, Access Keys, and IMDS

* If from an EC2 I want to access S3 via CLI or SDK, both IAM Roles and Access Keys will work, but using an IAM Role is the strongly recommended way.
* **Using an IAM Role (Best Practice):** You attach an IAM role to the EC2 instance (an Instance Profile). The AWS CLI and SDK automatically detect this and fetch temporary tokens from IMDS seamlessly.
* **Using Access Keys (Anti-pattern):** Hardcoding static keys into a `~/.aws/credentials` file on the EC2 server means if that server is compromised, those permanent keys are stolen.


* **Token Rotation:** For applications running inside AWS infrastructure (like an EC2 or a Lambda function), the AWS SDK handles token rotation automatically. The SDK talks to the **Instance Metadata Service (IMDS)** in the background. When it detects temporary credentials are near expiration, it silently fetches a brand new set of keys from IMDS.

### iam:PassRole & Cross-Delegation Scenarios

* The `iam:PassRole` permission allows an IAM identity to "assign" or attach an existing IAM role to an AWS service.
* *Example:* A Jenkins job uses an AWS role to create a job for AWS Batch. It will need to assign a role to that job, and for that, it will need `iam:PassRole` permission. You should scope this policy so it can only pass a specific role.
* **Jenkins Pipeline STS Delegation Example:**
1. Jenkins assumes a role that has permission for `sts:AssumeRole` (`jenkins-role`).
2. We have a role (`local-dev`) that has access to AWS services like S3.
3. Jenkins calls STS to get temporary creds for the `local-dev` role.
4. Developers copy the creds to their local machine to impersonate the `local-dev` role that was generated by Jenkins.


### AWS Organizations & IAM Identity Center (Multi-Account Setup)

* **AWS Organizations:** It is an AWS best practice to create multiple AWS accounts for different environments (e.g., Dev, Stg, Prod).
* An AWS account is the highest level of isolation. If a developer breaks something or runs up a massive bill in the "Dev" account, the "Prod" account remains completely safe and isolated.
* Organizations allow you to centrally manage billing and apply global security policies across all these accounts.


* **IAM Identity Center (Formerly AWS SSO):** This is the modern replacement for creating individual IAM users.
* *The Problem:* AWS has a limit of 5,000 IAM users per account. If a company has 10,000 employees, or if employees need to log into 15 different AWS accounts (Dev, QA, Prod, etc.), managing IAM users becomes impossible.
* *The Solution:* Identity Center connects to a company's existing corporate directory (like Microsoft Active Directory). Employees log in once (Single Sign-On) using their standard company credentials and see a portal of all the AWS accounts they are allowed to access.
* *Exam Rule:* If a scenario involves thousands of employees or multiple AWS accounts, **never** choose the option to "create IAM users." Always choose **Federation / Identity Center**.



---

### Permission Boundaries (The "Max Limit")

* A Permission Boundary is an advanced managed policy that sets the **maximum** permissions that an identity-based policy can grant to an IAM role or user.
* **A boundary does NOT grant permissions; it only limits them.**
* *The "Max Limit" Analogy:*
* Let's say I have a role (`R1`) and I attach a Permission Boundary to it that allows `CloudWatch` and `CloudTrail`.
* If `R1` currently only has an identity permission policy for `CloudWatch`, its effective permission is just `CloudWatch`.
* If I later add `CloudTrail` to `R1`'s identity policy, it works!
* However, if an admin later tries to add `S3` or `EC2` access to `R1`, the boundary acts as a brick wall and **blocks it**. `R1` will never have S3 or EC2 permission as long as that boundary is attached, regardless of what its identity policy says.


* **Common Use Case:** Allowing developers to create their own IAM roles (Delegated Administration) but forcing them to attach a Permission Boundary so they cannot accidentally or maliciously give themselves Administrator access (Privilege Escalation).

---

### Instance Profiles (EC2 Specifics)

* **What it is:** An Instance Profile is a "container" specifically designed to hold an IAM role and pass it to an Amazon EC2 instance.
* **Historical Context:** EC2 was created years before IAM. Because EC2 wasn't originally built to accept IAM roles natively, AWS created Instance Profiles as a bridge to deliver the role to the EC2 Instance Metadata Service (IMDS).
* **Current Status:** They are **NOT** deprecated. You still strictly need an Instance Profile to attach a role to an EC2 instance today.
* *Note:* When you attach a role to an EC2 instance via the AWS Console, AWS silently creates an Instance Profile with the exact same name in the background. However, if you automate infrastructure using the AWS CLI or CloudFormation, you must explicitly pass the **Instance Profile**, not just the Role ARN. Newer services (like Lambda) do not need Instance Profiles because they were built with native IAM integration.
