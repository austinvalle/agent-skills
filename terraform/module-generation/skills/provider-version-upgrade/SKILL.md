---
name: provider-version-upgrade
description: Step-by-step workflow for upgrading Terraform provider versions in module configurations with minimal risk. Use when bumping provider versions in required_providers, updating lock files, validating speculative plans, refactoring deprecated arguments, importing split resources, and handling provider regressions during upgrade.
metadata:
  copyright: Copyright IBM Corp. 2026
  version: "0.0.1"
---

# Provider Version Upgrade

Safely upgrade Terraform provider versions used by Terraform modules while keeping infrastructure behavior stable and predictable.

Source tutorial:
- https://developer.hashicorp.com/validated-patterns/terraform/upgrade-terraform-provider

## Use This Skill When

- Upgrading `required_providers` versions in module or root configuration.
- Reviewing upgrade pull requests for plan noise, deprecations, or drift.
- Refactoring configuration after provider deprecations.
- Importing newly split resources to avoid destructive or unnecessary changes.
- Troubleshooting unexpected plan changes after a provider bump.

## Core Principles

- Upgrade frequently in small increments.
- Isolate upgrade work in a dedicated branch and PR.
- Change one thing at a time and rerun `terraform plan` after each edit.
- Keep the first commit limited to provider version and lock file updates.
- Do not apply until plan output is understood and acceptable.
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

### 1. Prepare upgrade branch

```bash
git checkout -b upgrade-provider-<provider_name>-<to_version>
```

Capture current state before edits:

- Confirm Terraform CLI version compatibility.
- Run `terraform init` and baseline `terraform plan`.
- Record existing warnings or drift.

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

### 5. Import split resources instead of re-creating

When refactoring creates new Terraform resources for already-existing infrastructure, prefer configuration-driven import blocks (Terraform >= 1.5):

```hcl
import {
  to = aws_s3_bucket_versioning.this
  id = aws_s3_bucket.this.id
}
```

Run plan again and confirm actions are limited to import operations (or no-op where applicable).

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

### 7. Final verification and merge

Before merge:

- `terraform fmt -check`
- `terraform validate`
- `terraform plan` has no unexplained changes, warnings, or errors
- PR includes rationale, risk notes, and rollback strategy

After merge:

- Apply through your normal deployment workflow.
- Verify runtime behavior and monitoring.
- Remove temporary import blocks when no longer needed.

## Review Checklist

- Provider version upgrade is isolated from unrelated refactors.
- Lock file updates match intended providers/platforms.
- All deprecations are addressed or explicitly accepted with justification.
- New resources introduced for refactors are imported, not recreated.
- Any `ignore_changes` usage is temporary, narrow, and ticketed.
- Upgrade notes are captured for downstream module consumers.

## Anti-Patterns

- Skipping directly to `terraform apply` after changing provider versions.
- Mixing provider upgrade with large feature work in one PR.
- Ignoring warnings because plan says "No changes".
- Hiding unknown diffs with broad `ignore_changes` without issue tracking.
- Omitting `.terraform.lock.hcl` from version control.