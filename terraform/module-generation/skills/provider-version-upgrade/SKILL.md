---
name: provider-version-upgrade
description: Step-by-step workflow for upgrading Terraform provider versions in module configurations with minimal risk. Use when bumping provider versions in required_providers, updating lock files, validating speculative plans, refactoring deprecated arguments, and handling provider regressions during upgrade without modifying Terraform state.
metadata:
  copyright: Copyright IBM Corp. 2026
  version: "0.0.1"
---

# Provider Version Upgrade

Safely upgrade Terraform provider versions used by Terraform modules while keeping infrastructure behavior stable and predictable.

Task scope is limited to Terraform configuration files and `.terraform.lock.hcl` only. Configuration-driven `import` blocks are allowed when supported by Terraform 1.5+, because they remain reversible through configuration changes alone. Do not update Terraform state via CLI commands, do not run state-modifying commands such as `terraform apply` or `terraform import`, and do not make changes that require anything beyond reverting configuration and lock file edits.

Source tutorial:
- https://developer.hashicorp.com/validated-patterns/terraform/upgrade-terraform-provider

## Use This Skill When

- Upgrading `required_providers` versions in module or root configuration.
- Reviewing provider upgrade changes for plan noise, deprecations, or drift.
- Refactoring configuration after provider deprecations.
- Adding configuration-driven `import` blocks for split resources on Terraform 1.5+.
- Documenting follow-up state migration work when Terraform does not support configuration-driven imports or when CLI state changes would otherwise be required.
- Troubleshooting unexpected plan changes after a provider bump.

## Core Principles

- Upgrade frequently in small increments.
- Change one thing at a time and rerun `terraform plan` after each edit.
- Keep the first change limited to provider version and lock file updates.
- Limit all agent edits to Terraform configuration and `.terraform.lock.hcl`.
- Configuration-driven `import` blocks are allowed only when Terraform supports them.
- Never run commands that modify Terraform state.
- Every agent change must be reversible by undoing the configuration and lock file edits.
- Treat provider bugs as first-class incidents with documented workarounds.

## Inputs

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `provider_name` | string | Yes | Provider local name (for example, `aws`, `azurerm`, `google`) |
| `provider_source` | string | Yes | Provider source address (for example, `hashicorp/aws`) |
| `from_version` | string | Yes | Current provider version or constraint |
| `to_version` | string | Yes | Target provider version or constraint |
| `module_paths` | list(string) | No | Module paths to scan and update |
| `workspace_context` | string | No | Local CLI, HCP Terraform, or Terraform Enterprise context |

## Workflow

### 1. Prepare the upgrade context

Capture current state before edits:

- Confirm Terraform CLI version compatibility.
- Run `terraform init` and baseline `terraform plan`.
- Record existing warnings or drift.
- Confirm the task does not require state changes to complete.

### 2. Update provider version only

Change only `required_providers` constraints first.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Install upgraded provider and refresh lock file:

```bash
terraform init -upgrade
```

Expected file changes in this step:

- Version constraint files (for example, `versions.tf`).
- `.terraform.lock.hcl`.

### 3. Run speculative plan and triage

```bash
terraform plan
```

Classify output into:

- Errors: must be fixed before proceeding.
- Warnings: often deprecations requiring refactors.
- Actions (`add`, `change`, `destroy`, `import`): validate whether expected.

Document every warning/action before making edits.

### 4. Refactor incrementally

For each warning or behavior change:

- Read provider upgrade guide and resource docs.
- Make one minimal refactor.
- Re-run `terraform plan`.

Common pattern for deprecations: move inline nested arguments into dedicated resources.

Example migration shape:

```hcl
# Old
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  versioning {
    enabled = true
  }
}

# New
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

### 5. Add import blocks when Terraform supports them

When refactoring creates new Terraform resources for already-existing infrastructure and the workspace uses Terraform 1.5 or later, add configuration-driven `import` blocks instead of relying on CLI state changes:

```hcl
import {
  to = aws_s3_bucket_versioning.this
  id = aws_s3_bucket.this.id
}
```

After adding import blocks:

- Re-run speculative `terraform plan`.
- Confirm the remaining actions are understood and acceptable.
- Keep the import blocks in configuration only; do not run `terraform import` or `terraform apply`.

If Terraform is older than 1.5, or if the migration still requires CLI-driven state movement, stop the implementation and document the exact resource addresses and expected follow-up action instead.

### 6. Handle provider regressions safely

If plan shows known provider bugs:

- Search upstream provider issues and release notes.
- Apply temporary, scoped workarounds only when necessary.
- Add a tracking ticket to remove workaround after provider fix.

Temporary workaround pattern:

```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  lifecycle {
    ignore_changes = [
      server_side_encryption_configuration
    ]
  }
}
```

### 7. Final verification and handoff

Before handing the change back to the caller:

- `terraform fmt -check`
- `terraform validate`
- `terraform plan` has no unexplained changes, warnings, or errors
- Handoff notes include rationale, risk notes, and rollback strategy

Out of scope for this skill:

- Applying the change.
- CLI-driven importing or moving state.
- Any workflow whose effects cannot be fully reverted by undoing configuration and lock file edits.

## Review Checklist

- Provider version upgrade is isolated from unrelated refactors.
- Lock file updates match intended providers/platforms.
- All deprecations are addressed or explicitly accepted with justification.
- Agent edits are limited to Terraform configuration and `.terraform.lock.hcl`.
- Any required CLI-driven state migration is documented, not implemented.
- Configuration-driven `import` blocks are used only when Terraform 1.5+ supports them.
- Any `ignore_changes` usage is temporary, narrow, and ticketed.
- Upgrade notes are captured for downstream module consumers.

## Anti-Patterns

- Skipping directly to `terraform apply` after changing provider versions.
- Mixing provider upgrade with large, unrelated changes in the same task.
- Ignoring warnings because plan says "No changes".
- Running `terraform apply`, `terraform import`, or any other state-changing command.
- Using CLI state migration when configuration-driven `import` blocks would suffice.
- Hiding unknown diffs with broad `ignore_changes` without issue tracking.
- Omitting `.terraform.lock.hcl` from version control.