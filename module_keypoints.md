# Terraform Modules — Real-Time Structure, Locals vs Variables, Outputs & Common Errors

Reference notes covering how modules are actually structured in real projects, why `locals` can never be overridden from outside, how environments are declared cleanly, how values flow between modules via outputs, and the exact errors Terraform throws when a variable contract is broken.

---

## 1. Real-Time Module Structure

In real projects, Terraform is split into **reusable child modules** (generic, environment-agnostic) and **root modules per environment** (glue + environment-specific values).

```
repo/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── locals.tf
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── locals.tf
│   └── iam/
│       └── ...
│
└── environments/
    ├── dev/
    │   ├── main.tf          # calls modules/vpc, modules/eks
    │   ├── variables.tf
    │   ├── dev.tfvars
    │   └── backend.hcl
    ├── test/
    │   ├── main.tf
    │   ├── test.tfvars
    │   └── backend.hcl
    └── prod/
        ├── main.tf
        ├── prod.tfvars
        └── backend.hcl
```

**Key real-time practices:**

| Practice | Why |
|---|---|
| One `modules/` folder, reused across envs | Single source of truth — fix a bug once, applies everywhere. |
| One root module (folder) per environment | Each env gets its **own state file**, own backend, own blast radius. |
| Module sourced via Git with `?ref=<tag/branch>` | Pins module version; `ref=v1.2.0` for prod, `ref=main` for dev. |
| Separate `.tfvars` per environment | No hardcoded values inside modules; values injected per env. |
| Remote backend per environment (S3 + DynamoDB, etc.) | State isolation between dev/test/prod — a mistake in dev can't corrupt prod state. |

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"
  environment = var.environment
  cidr_block  = var.vpc_cidr
}
```

---

## 2. Locals Cannot Be Overridden

`locals` are **computed, internal values** — not inputs. They exist only to avoid repeating an expression inside the module where they're declared.

```hcl
# modules/eks/locals.tf
locals {
  cluster_name = "${var.environment}-app-cluster"
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

**Why they can't be overridden from a parent module:**

- A `locals` block has **no external interface** — Terraform never exposes `locals` as something a caller can set. Only `variable` blocks accept external input (via the `module "x" { ... }` block).
- `locals` are scoped strictly to the module they're declared in. A root module cannot reach into `module.eks`'s locals, and `module.eks` cannot see the root's locals or a sibling module's locals.
- Locals can **read** variables (`local.x = var.y`), but the dependency only flows one direction: `variable → local`, never `local → variable`.
- If you need a value to be configurable from outside, it **must** be a `variable`, not a `local`. Trying to pass `local_name = "value"` into a module block simply throws `Unsupported argument`, because Terraform only recognizes declared `variable` names as valid module arguments.

**Rule of thumb:** `variable` = the module's public interface (inputs). `output` = the module's public interface (return values). `locals` = private implementation detail, invisible outside the module.

---

## 3. How Environments Are Declared

Real-time projects avoid relying on `terraform.workspace` alone for full environment separation (it doesn't isolate backends/state storage well at scale). The standard pattern:

1. **Declare the variable** at root level:
   ```hcl
   variable "environment" {
     type        = string
     description = "Deployment environment (dev/test/prod)"
   }
   ```
2. **Supply the value** via a per-environment `.tfvars` file:
   ```hcl
   # dev.tfvars
   environment = "dev"
   ```
3. **Run with the matching var file** (usually wired into CI/CD per branch or pipeline stage):
   ```bash
   terraform plan  -var-file="dev.tfvars"
   terraform apply -var-file="dev.tfvars"
   ```
4. **Pass it down into every child module** explicitly:
   ```hcl
   module "eks" {
     source      = "../../modules/eks"
     environment = var.environment
   }
   ```
5. **Backend isolation per environment** (separate state/lock table per env), typically via partial backend config:
   ```bash
   terraform init -backend-config="backend.hcl"
   ```

This keeps one codebase, N environments, N isolated states — and `var.environment` becomes the single value that drives naming, tagging, and sizing differences across the whole stack.

---

## 4. Getting a Value/ID/Name from Another Module — Output Values

Modules are isolated — `module.vpc` cannot directly reference a resource inside `module.eks`, and vice versa. The only way data crosses a module boundary is **output → input**, and it must pass **through the calling (root) module**.

**Step 1 — Child module exposes the value as an output:**

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

**Step 2 — Root module references it using `module.<name>.<output>`:**

```hcl
# environments/dev/main.tf
module "vpc" {
  source = "../../modules/vpc"
  ...
}

module "eks" {
  source     = "../../modules/eks"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

**Key points:**

- `module.vpc.vpc_id` only works in the **caller's** scope (root module, or whichever module declared `module "vpc" { ... }`). Sibling modules never see each other directly.
- Terraform automatically builds the dependency graph from this reference — `module.eks` will implicitly wait for `module.vpc` to finish creating resources, no `depends_on` needed for this case.
- For **cross-state** references (e.g., networking managed in a completely separate Terraform run/pipeline), use `terraform_remote_state` instead:
  ```hcl
  data "terraform_remote_state" "vpc" {
    backend = "s3"
    config = {
      bucket = "tf-state-bucket"
      key    = "dev/vpc/terraform.tfstate"
      region = "us-west-2"
    }
  }
  # usage: data.terraform_remote_state.vpc.outputs.vpc_id
  ```

---

## 5. Variable Declared in Module but Not Declared/Passed by Consumer

This depends on **where the gap is** — three distinct scenarios, three distinct errors:

### Scenario A — Module declares a required variable (no default), consumer doesn't pass it
```hcl
# modules/eks/variables.tf
variable "environment" {
  type = string          # no default → required
}
```
```hcl
# root main.tf — environment argument omitted
module "eks" {
  source = "../../modules/eks"
}
```
**Result:**
```
Error: Missing required argument
The argument "environment" is required, but no definition was found.
```

### Scenario B — Module declares a variable with a default, consumer doesn't pass it
```hcl
variable "environment" {
  type    = string
  default = "dev"
}
```
**Result:** No error — Terraform silently uses the default. This is the safe pattern for optional knobs.

### Scenario C — Consumer passes an argument the module never declared
```hcl
module "eks" {
  source      = "../../modules/eks"
  environment = var.environment   # module has no `variable "environment"`
}
```
**Result:**
```
Error: Unsupported argument
An argument named "environment" is not expected here.
```
This is the inverse of Scenario A — the module's interface simply doesn't recognize that name (common cause: value exists as a `local` instead of a `variable` in the child module — see Section 2).

### Scenario D — Root-level variable has no default and no value supplied anywhere
```hcl
variable "environment" {
  type = string
}
```
No `.tfvars`, no `-var`, no `TF_VAR_environment` env var set.
**Result:**
- **Interactive run:** Terraform prompts: `var.environment  Enter a value:`
- **Non-interactive run (CI/CD):** 
  ```
  Error: No value for required variable
  The root module input variable "environment" is not set, and has no default value.
  ```

### Summary Table

| Scenario | Cause | Error |
|---|---|---|
| Required child variable, not passed | Missing module argument | `Missing required argument` |
| Optional child variable (has default), not passed | — | No error, default used |
| Consumer passes an argument child never declared | Name mismatch / used `local` instead of `variable` | `Unsupported argument` |
| Root variable required, no value anywhere, non-interactive | No `.tfvars`/`-var`/env var | `No value for required variable` |
| Root variable required, no value anywhere, interactive | Same as above | Terraform prompts interactively |

---

## Core Takeaways

- **`variable`** = configurable input, crosses module boundaries. **`local`** = private, computed, never overridable from outside.
- Environments are just a `var.environment` string flowing from `.tfvars` → root → every child module, paired with per-env backend isolation.
- Cross-module data always flows **output → root → input** of the next module; sibling modules never talk to each other directly.
- Every variable/argument mismatch error boils down to one rule: **the child module's `variables.tf` is its contract** — the parent can only pass what's declared, and must pass whatever has no default.