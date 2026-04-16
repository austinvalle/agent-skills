---
name: module-version-upgrade
description: Step-by-step workflow for upgrading Terraform module versions in configuration with minimal risk. Use when bumping module version arguments, reconciling module-to-provider constraint mismatches, validating speculative plans, refactoring configuration for module internals that changed, and handling provider bugs encountered during module upgrades without modifying Terraform state.
metadata:
  copyright: Copyright IBM Corp. 2026
  version: "0.0.1"
---

# Module Version Upgrade

Safely upgrade Terraform module versions while preserving stable infrastructure behavior and a predictable rollout process.

Task scope is limited to Terraform configuration files and `.terraform.lock.hcl` only. Configuration-driven `import` blocks are allowed when supported by Terraform 1.5+, because they remain reversible through configuration changes alone. Do not update Terraform state via CLI commands, do not run state-modifying commands such as `terraform apply` or `terraform import`, and do not make changes that require anything beyond reverting configuration and lock file edits.

Source tutorial:
- https://developer.hashicorp.com/validated-patterns/terraform/upgrade-and-refactor-terraform-modules

## Use This Skill When

- Upgrading `version` in Terraform `module` blocks.
- Reviewing module upgrade changes for unexpected plan actions.
- Resolving provider constraints introduced by new module versions.
- Refactoring configuration when a module release changes underlying resource patterns.
- Adding configuration-driven `import` blocks for split resources on Terraform 1.5+.
- Documenting follow-up state migration work when Terraform does not support configuration-driven imports or when CLI state changes would otherwise be required.

## Core Principles

- Upgrade modules frequently, ideally near release.
- Keep the first change limited to the module version only.
- Limit all agent edits to Terraform configuration and `.terraform.lock.hcl`.
- Configuration-driven `import` blocks are allowed only when Terraform supports them.
- Use speculative plans continuously during refactoring.
- Address one warning/error/action at a time.
- Never run commands that modify Terraform state.
- Every agent change must be reversible by undoing the configuration and lock file edits.

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

### 1. Prepare the upgrade context

Before editing:

- Confirm baseline `terraform plan` is clean or known.
- Record Terraform CLI version in use.
- Identify impacted workspaces/environments.
- Confirm the task does not require state changes to complete.

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

Capture findings in handoff notes so each remediation is traceable.

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

### 6. Add import blocks when Terraform supports them

If speculative plan shows a resource add/change for infrastructure that already exists and the workspace uses Terraform 1.5 or later, add configuration-driven `import` blocks instead of relying on CLI state changes:

```hcl
import {
  to = module.s3_bucket.aws_s3_bucket_server_side_encryption_configuration.this[0]
  id = var.bucket_name
}
```

After adding import blocks:

- Re-run speculative `terraform plan`.
- Confirm the remaining actions are understood and acceptable.
- Keep the import blocks in configuration only; do not run `terraform import` or `terraform apply`.

If Terraform is older than 1.5, or if the migration still requires CLI-driven state movement, stop the implementation and document the exact resource addresses and expected follow-up action instead.

### 7. Handle provider regressions discovered during module upgrade

If unexpected drift remains after following module upgrade guidance:

- Search provider issue tracker and module issues.
- Prefer moving to an unaffected provider version when feasible.
- Document temporary mitigations and follow-up cleanup tasks.

### 8. Final verification and handoff

Before handing the change back to the caller:

- `terraform fmt -check`
- `terraform validate`
- `terraform plan` with no unexplained actions
- Handoff notes explain the module bump, provider impacts, any out-of-scope state migration follow-up, and risk

Out of scope for this skill:

- Applying the change.
- CLI-driven importing or moving state.
- Any workflow whose effects cannot be fully reverted by undoing configuration and lock file edits.

## Review Checklist

- Module version bump is isolated from unrelated changes.
- Upstream module changelog/upgrade guide was reviewed.
- Provider constraints updated only when required by module version.
- Agent edits are limited to Terraform configuration and `.terraform.lock.hcl`.
- Any required CLI-driven state migration is documented, not implemented.
- Configuration-driven `import` blocks are used only when Terraform 1.5+ supports them.
- Remaining actions in speculative plan are explained and approved.
- Rollback path is documented.

## Anti-Patterns

- Upgrading module and provider blindly in one large change set without plan checkpoints.
- Ignoring `terraform init -upgrade` errors about provider constraints.
- Running `terraform apply`, `terraform import`, or any other state-changing command.
- Using CLI state migration when configuration-driven `import` blocks would suffice.
- Applying with unexplained `add`, `change`, or `destroy` actions.
- Skipping module-specific upgrade guides/changelogs.
- Leaving temporary mitigations undocumented.