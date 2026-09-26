# Terraform Module Variables & EKS Partial Apply

Notes on two Terraform issues: how child module input variables work, and how to recover from a partial EKS apply where the cluster exists in AWS but not in Terraform state.

---

## 1. Terraform Module Variables

When a mandatory variable is not declared in the child module, Terraform throws an error because it cannot find the required variable.

### What We Learned

| Concept | What we learned |
|---|---|
| Module variable | A child module must declare an input using `variable "environment"` before the parent can pass `environment = ...`. |
| `locals.tf` | `environment = var.environment` inside locals only creates a local value/tag. It does not declare a module input variable. |
| Case-sensitive | Terraform variable and module argument names are case-sensitive. `environment` and `Environment` are different. |
| Spelling matters | `environment` and `environemnt` are completely different names. |
| Parent → Child | The parent module passes the value to the reusable module using `environment = var.environment`. |
| Child module | The child module receives the value through `variable "environment"`. |
| `terraform.tfvars` | Provides the actual value, for example `environment = "dev"`. |
| Module reuse | The same VPC/EKS module can be reused for Dev, Test and Prod by passing different variable values. |
| Git module source | Using `?ref=main` downloads the module from the main branch of GitHub. |
| Module refresh | `terraform init -upgrade` forces Terraform to check for updated module versions. |

### Interview Point

Terraform module variables must be explicitly declared in the child module. Defining a key inside `locals.tf` does not make it a module input. The parent module passes the value using the exact variable name declared by the child module.

### Common Interview Question

**Q:** I have `environment = var.environment` in my module block, but Terraform says `Unsupported argument`. Why?

**A:** The child module does not expose an input variable with that exact name. I would check the child module's `variables.tf` and verify that `variable "environment"` is declared with the exact spelling and case.

### Terraform Variable Flow

```
terraform.tfvars
       ↓
Root variable
       ↓
Module block
       ↓
Child module variable
       ↓
Child resources / locals
```

---

## 2. EKS Terraform Partial Apply

Started creating an EKS environment containing approximately 43 resources:

- S3
- VPC
- EKS
- IAM
- Add-ons
- Node Groups
- Other supporting resources

The Terraform apply was interrupted after creating some resources.

### Initial State

- **Total resources planned:** 43
- **`terraform state list`:** 32 resources present
- **`terraform plan`:** `Plan: 17 to add, 0 to change, 0 to destroy`

The EKS cluster was visible and `ACTIVE` in AWS, but the Application and System node groups had not been created.

Running `terraform apply` again produced:

```
Error: creating EKS Cluster (dev-app-cluster):
operation error EKS: CreateCluster,
StatusCode: 409,
ResourceInUseException:
Cluster already exists with name: dev-app-cluster
```

### Investigation

First, verified the EKS cluster:

```bash
aws eks describe-cluster \
  --name dev-app-cluster \
  --region us-west-2 \
  --query 'cluster.{Name:name,Status:status,Version:version,Endpoint:endpoint}' \
  --output table
```

Result:

```
Name     : dev-app-cluster
Status   : ACTIVE
Version  : 1.33
```

Then checked Terraform state:

```bash
terraform state list | grep -i eks
```

The IAM roles and other EKS-related resources were present, but the actual EKS cluster resource was missing:

```
module.eks.aws_iam_role.cluster
module.eks.aws_iam_role.node
...
```

There was **no** `module.eks.aws_eks_cluster.main`.

### Root Cause

The EKS cluster had already been created in AWS, but Terraform did not have the cluster in its state.

```
AWS
└── dev-app-cluster          ✅

Terraform State
└── EKS cluster              ❌
```

Terraform therefore attempted to create the cluster again, resulting in the `409 ResourceInUseException`.

### Solution

Imported the existing EKS cluster into Terraform state:

```bash
terraform import module.eks.aws_eks_cluster.main dev-app-cluster
```

After importing, `terraform plan` still showed the cluster being replaced. The key difference was:

```
bootstrap_cluster_creator_admin_permissions = true -> null
# forces replacement
```

The existing AWS cluster had this setting enabled, but it was not defined in the Terraform configuration.

Updated the EKS cluster configuration:

```hcl
access_config {
  authentication_mode                         = "API_AND_CONFIG_MAP"
  bootstrap_cluster_creator_admin_permissions = true
}
```

Then:

```bash
terraform init -upgrade
terraform plan
```

### Final Plan

The cluster was no longer marked for replacement. Terraform showed:

```
# module.eks.aws_eks_cluster.main will be updated in-place
```

with the relevant change being tags:

```
~ tags = {
    + "Environment"            = "dev"
    + "ManagedBy"              = "Terraform"
      "Name"                   = "dev-app-cluster"
      "Terraform"              = "true"
      "karpenter.sh/discovery" = "dev-app-cluster"
  }
```

The EKS cluster was therefore successfully reconciled with Terraform state **without destroying and recreating the existing cluster**.