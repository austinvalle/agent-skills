---
name: module-version-upgrade
description: Step-by-step workflow for upgrading Terraform module versions in configuration with minimal risk. Use when bumping module version arguments, reconciling module-to-provider constraint mismatches, validating speculative plans, refactoring for module internals that changed, importing newly surfaced resources, and handling provider bugs encountered during module upgrades.
metadata:
  copyright: Copyright IBM Corp. 2026
  version: "0.0.1"
---

# Module Version Upgrade

Safely upgrade Terraform module versions while preserving stable infrastructure behavior and a predictable rollout process.

Source tutorial:
- https://developer.hashicorp.com/validated-patterns/terraform/upgrade-and-refactor-terraform-modules

## Use This Skill When

- Upgrading `version` in Terraform `module` blocks.
- Reviewing module upgrade PRs for unexpected plan actions.
- Resolving provider constraints introduced by new module versions.
- Refactoring configuration when a module release changes underlying resource patterns.
- Importing resources to align state after module internals change.

## Core Principles

- Upgrade modules frequently, ideally near release.
- Isolate each module upgrade in its own branch and PR.
- Keep the first change to module version only.
- Use speculative plans continuously during refactoring.
- Address one warning/error/action at a time.
- Apply only after plan output is fully understood and acceptable.

## Inputs

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `module_name` | string | Yes | Module block label to upgrade |
| `module_source` | string | Yes | Module source address (registry, Git, or local path) |
| `from_version` | string | Yes | Current module version |
| `to_version` | string | Yes | Target module version |
| `root_module_path` | string | Yes | Root configuration path where module is consumed |
| `provider_set` | list(string) | No | Providers used by the upgraded module |

## Workflow

### 1. Prepare upgrade branch

```bash
git checkout -b upgrade-module-<module_name>-<to_version>
```

Before editing:

- Confirm baseline `terraform plan` is clean or known.
- Record Terraform CLI version in use.
- Identify impacted workspaces/environments.

### 2. Upgrade module version only

Change only the module `version` argument first.

```hcl
module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "3.0.0"

  bucket               = var.bucket_name
  attach_public_policy = false
}
```

Re-initialize and upgrade dependencies:

```bash
terraform init -upgrade
```

### 3. Identify and document upgrade impacts

Classify output from `terraform init -upgrade` and `terraform plan`:

- Version constraint errors (often provider mismatch).
- Warnings/deprecations from updated module behavior.
- Actions (`add`, `change`, `destroy`, `import`) in speculative plan.

Capture findings in a ticket/PR notes so each remediation is traceable.

### 4. Resolve module-to-provider mismatches (handoff)

If `terraform init -upgrade` or `terraform plan` indicates provider constraint mismatches after a module bump, hand off provider remediation to the dedicated skill:

- `terraform/module-generation/skills/provider-version-upgrade/SKILL.md`

Use this skill as the source of truth for provider-specific decisions, lock file handling, deprecation refactors, and provider bug mitigations.

Return to this workflow once provider upgrade steps are complete and resume with speculative plan review for module-specific issues.

### 5. Refactor for changed module internals

New module versions can split behavior into additional resources. Even if behavior is equivalent, plan may show adds/changes.

For each issue:

- Check module `CHANGELOG` and upgrade guide.
- Make one targeted change.
- Re-run speculative plan.

### 6. Import resources introduced by module changes

When plan shows a resource add that already exists in cloud reality, use import blocks (Terraform >= 1.5):

```hcl
import {
  to = module.s3_bucket.aws_s3_bucket_server_side_encryption_configuration.this[0]
  id = var.bucket_name
}
```

Expected outcome is no `add`/`destroy` actions; imports are acceptable during migration.

### 7. Handle provider regressions discovered during module upgrade

If unexpected drift remains after following module upgrade guidance:

- Search provider issue tracker and module issues.
- Prefer moving to an unaffected provider version when feasible.
- Document temporary mitigations and follow-up cleanup tasks.

### 8. Final verification and rollout

Before merge:

- `terraform fmt -check`
- `terraform validate`
- `terraform plan` with no unexplained actions
- PR explains module bump, provider impacts, imports, and risk

After merge:

- Apply via normal CI/CD workflow.
- Verify deployment and runtime behavior.
- Remove one-time import blocks if team policy requires cleanup.

## Review Checklist

- Module version bump is isolated from unrelated changes.
- Upstream module changelog/upgrade guide was reviewed.
- Provider constraints updated only when required by module version.
- Imports are used for existing resources to avoid unnecessary recreate.
- Remaining actions in speculative plan are explained and approved.
- Rollback path is documented.

## Anti-Patterns

- Upgrading module and provider blindly in one large commit without plan checkpoints.
- Ignoring `terraform init -upgrade` errors about provider constraints.
- Applying with unexplained `add`, `change`, or `destroy` actions.
- Skipping module-specific upgrade guides/changelogs.
- Leaving temporary mitigations undocumented.