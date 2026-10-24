# AWS Organizations – Service Control Policies (SCPs)

## 1. What is an SCP?

A **Service Control Policy (SCP)** is a policy in AWS Organizations that defines the maximum permissions available to IAM users and IAM roles in member accounts.

- SCPs are used to enforce organization-wide security guardrails.
- They do not grant permissions.
- IAM policies must still grant the required permissions.
- SCPs can restrict actions even when an IAM policy allows them.

## 2. Where do SCPs reside?

SCPs are created and managed in **AWS Organizations**, typically from the management account.

```text
AWS Organizations
       |
      Root
       |  Root-level SCP
       |
   Production OU
       |  Production SCP
       |
   Production Account
       |  Inherits applicable SCPs
       |
   IAM Users / IAM Roles
       |
   AWS Resources
   (EC2, S3, RDS, EKS)
```

SCPs can be attached to:
1. Organization root
2. Organizational units (OUs)
3. Individual AWS accounts

Applicable SCPs from parent levels are inherited by child OUs and accounts.

## 3. What rules are commonly configured?

| SCP rule | Purpose |
|---|---|
| Region restriction | Restrict operations in unapproved AWS Regions. |
| Security protection | Prevent selected security controls from being disabled. |
| Organization protection | Prevent member accounts from leaving the organization. |
| Resource protection | Block selected high-risk creation, modification, or deletion actions. |
| Production guardrails | Restrict destructive operations in production accounts. |

The rules depend on the organization's security and compliance requirements.

## 4. Example SCP – Deny EC2 Instance Creation

This example blocks launching EC2 instances in affected member accounts.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEC2Creation",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "*"
    }
  ]
}
```

Even if an IAM policy allows `ec2:RunInstances`, this applicable explicit deny prevents the operation.

## 5. Does an SCP restrict user-level access?

**Yes.** SCPs restrict the permissions available to IAM users and IAM roles in affected member accounts.

Example:

- IAM policy allows `s3:*`.
- SCP denies `s3:DeleteBucket`.
- Result: the user or role cannot delete S3 buckets, but other S3 actions may remain possible if allowed by applicable permissions.

SCPs apply to the member account's root user as well, but they do not restrict principals in the AWS Organizations management account. Service-linked roles are also exempt from SCP restrictions.

## 6. SCP vs IAM Policy

| Feature | SCP | IAM policy |
|---|---|---|
| Main purpose | Organization-wide guardrails | Grant or restrict permissions |
| Managed through | AWS Organizations | AWS IAM |
| Attached to | Root, OU, or account | IAM users, groups, roles, or resources, depending on policy type |
| Grants permissions? | No | Identity-based allow policies can grant permissions |
| Can block an action despite an IAM allow? | Yes | An explicit deny can also block an action |

## 7. Real-Time Interview Answer

"In our AWS enterprise environment, we use SCPs in AWS Organizations to enforce centralized security guardrails across multiple AWS accounts. We attach SCPs at the root, OU, or account level. For example, we can restrict unapproved Regions, prevent selected security controls from being disabled, and block high-risk operations in production. SCPs limit the maximum permissions available to IAM users and roles in member accounts, while IAM policies grant the actual permissions. An SCP does not grant access by itself."

## 8. Important Points to Remember

- SCPs are managed centrally in AWS Organizations.
- SCPs do not grant permissions.
- Applicable SCPs and IAM permissions work together.
- An explicit deny in an applicable SCP blocks the action.
- SCPs affect member accounts, including their root users.
- SCPs do not restrict the management account or service-linked roles.
- Test SCPs in a non-production OU before applying them broadly.
