# Real-Time AWS Multi-Account Architecture

## 1. Architecture Overview

A company uses **AWS Organizations** to manage multiple AWS accounts, **LDAP/Active Directory** for corporate identities, **IAM Identity Center** for centralized AWS access, and **Service Control Policies (SCPs)** for organization-wide guardrails.

```text
                 Corporate Users
                       |
                LDAP / Active Directory
                       |
             IAM Identity Center
             (Users, Groups, MFA)
                       |
                 Permission Sets
                       |
                 AWS Organizations
                       |
          Organization Root / OUs
                       |
       +---------------+----------------+
       |               |                |
 Management        DevOps OU       Workloads OU
 Account               |                |
                  +----+----+       +----+----+
                  |         |       |         |
               DevOps    Shared   Developer  SRE
               Account   Services Account    Account
                         Account

 SCPs = Guardrails attached to Root, OUs, or member accounts
```

*Note: This is a simplified example. In a production environment, OUs are usually designed around workload environments and common security controls, rather than only team names.*

## 2. Purpose of Each Component

| Component | Real-time purpose |
|---|---|
| LDAP / Active Directory | Stores corporate users, groups, and identity information. |
| IAM Identity Center | Provides centralized sign-in and AWS account access. |
| Permission sets | Define the AWS permissions assigned to users or groups. |
| AWS Organizations | Centrally manages multiple AWS accounts and OUs. |
| SCPs | Set maximum permissions and enforce organizational guardrails. |
| IAM roles and policies | Grant actual permissions to users, applications, and automation. |
| CloudTrail | Records AWS API activity for auditing. |

## 3. How LDAP Connects to IAM Identity Center

1. The company maintains employees in LDAP/Active Directory.
2. IAM Identity Center is configured with a supported identity source, such as an external identity provider or Active Directory.
3. Users and groups are made available to IAM Identity Center. The integration method determines how authentication and provisioning work.
4. The AWS administrator assigns groups to AWS accounts through permission sets.
5. Employees sign in through the AWS access portal and receive access to their assigned accounts.

**Important:** LDAP is not automatically connected to AWS. The organization must configure a supported integration. LDAP itself is a directory protocol; Microsoft Active Directory is a directory service.

## 4. Example Group and Account Assignments

| Group | Account | Permission set |
|---|---|---|
| DevOps | DevOps account | DevOps PowerUser |
| Developers | Developer account | Developer Access |
| SRE | SRE account | SRE Operations |
| Platform Team | Shared Services account | Platform Admin |
| Cloud Security | Security accounts, if configured | Security Audit |

Use least privilege: avoid giving every group administrator access. Keep management account access restricted to the small number of people who need it.

## 5. What SCPs Do in Real Time

SCPs are attached through AWS Organizations to the organization root, OUs, or individual member accounts. SCPs inherited from parent levels also apply.

Common examples:

- Prevent member accounts from disabling required security controls.
- Restrict use of unapproved AWS Regions.
- Prevent member-account users and roles from leaving the organization.
- Restrict certain high-risk services or actions.
- Establish guardrails for production accounts.

**Critical interview point:** An SCP does not grant permissions. It limits the maximum permissions available. An IAM policy or another applicable permission policy must still grant the action, and all applicable restrictions must permit it.

For example, if an IAM policy allows an engineer to create EC2 instances but an applicable SCP denies `ec2:RunInstances`, the request is denied.

SCPs do not restrict principals in the AWS Organizations management account, so workloads should generally run in member accounts.

## 6. How a Developer Accesses AWS

1. The employee signs in using the company's configured identity source.
2. IAM Identity Center authenticates the user and evaluates the account assignments.
3. The user selects the Developer AWS account and obtains a session for the assigned permission set.
4. IAM permissions determine which operations are allowed.
5. SCPs and other applicable controls limit those permissions.
6. CloudTrail records supported AWS API activity for auditing.

For local command-line access, the engineer can use AWS CLI IAM Identity Center profiles:

```bash
aws configure sso
aws sso login --profile developer
aws sts get-caller-identity --profile developer
```

Terraform can then use the selected profile:

```bash
export AWS_PROFILE=developer
terraform plan
```

The profile name and assigned account access must match the actual configuration.

## 7. How GitLab CI/CD Accesses AWS

GitLab CI/CD usually follows a separate machine-identity path:

```text
GitLab Pipeline
      |
      | OIDC token
      v
AWS IAM OIDC Provider
      |
      v
IAM Role with Trust Policy
      |
      v
Temporary AWS Credentials
      |
      v
Terraform Deployment
      |
      v
Target AWS Account
```

- Configure the GitLab OIDC provider and an IAM role trust policy.
- Restrict the trust policy to the intended project and branch or other approved claims.
- Grant the role only the IAM permissions required for deployment.
- For multiple target accounts, use controlled cross-account role assumption where appropriate.
- Applicable SCPs still restrict the role's actions in member accounts.
- Use temporary credentials rather than long-lived AWS access keys.

## 8. Interview Summary

“In our real-time AWS environment, we use AWS Organizations to manage multiple accounts. Corporate users are maintained in LDAP or Active Directory and integrated with IAM Identity Center. We assign groups to AWS accounts using permission sets, so employees get centralized, role-based access. We use SCPs at the organization root or OU level to enforce guardrails, while IAM policies grant the actual permissions. Developers use IAM Identity Center for local AWS CLI and Terraform access, while GitLab pipelines use OIDC to assume IAM roles and deploy using temporary credentials. CloudTrail provides audit visibility.”

Official AWS references
- AWS Organizations multi-account best practices
- AWS Service Control Policies (SCPs)
- AWS Organizations management-account best practices