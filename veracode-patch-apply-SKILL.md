---
name: veracode-patch-apply
description: Safely review and apply previously generated Veracode remediation patches after explicit developer approval.
---

# Veracode Patch Apply Skill

## 1. Purpose

This skill is the controlled second phase of remediation.

It applies only patch files already generated under:

`.GitHub/remediation/veracode/`

It must not discover or invent new vulnerabilities.

## 2. Select patch set

Discover project/module directories immediately under `.GitHub/remediation/veracode`.

Select the requested project/module when supplied. Otherwise select automatically only when there is exactly one valid choice; ask when there are multiple choices.

Choose the latest valid remediation date unless the user explicitly selects another one.

A valid patch set contains one or more `.patch` files and preferably `selected-scan.json` and `remediation-report.md`.

## 3. Pre-apply checks

Before asking for approval:

1. list all patch files;
2. extract each repository-relative target source path;
3. verify every target source file exists;
4. inspect Git status;
5. detect pre-existing modifications to affected files;
6. perform a non-destructive dry-run/check for each patch when available.

If an affected file has unrelated uncommitted changes, warn the developer and do not apply until explicitly resolved.

If any patch no longer matches its target source, stop and report that the patch must be regenerated or manually reviewed.

## 4. Approval gate

Display a concise summary and ask exactly:

`Apply these Veracode remediation patches to the source files? [Y/N]:`

Only explicit `Y` or `Yes` authorizes source modification.

Any other response means stop without changes.

## 5. Rollback protection

Before applying an approved patch set:

- capture `git status`;
- capture the current diff for affected files when applicable;
- preserve enough information to restore the pre-apply state;
- do not overwrite unrelated developer work.

Never create an automatic Git commit.

## 6. Apply patches

Apply patches in a deterministic order.

For each patch:

1. verify its target path;
2. apply it;
3. record success/failure;
4. stop on unsafe conflicts or malformed patches.

Do not guess through failed hunks.

## 7. Validation

After all patches are applied, run the narrowest suitable validation for the affected project/module, for example:

- Maven: targeted compile/test for the affected module;
- Gradle: targeted compile/test;
- Angular/Node: build, test, lint, or type-check as appropriate;
- Mainframe/COBOL: available syntax/compiler validation when configured.

Do not run an unnecessarily broad repository build when a module-level validation is sufficient.

## 8. Failure handling

If application or validation fails:

- do not commit;
- report the failing patch/command;
- preserve rollback information;
- offer to restore the affected files to the pre-apply state.

Do not mark failed changes as remediated.

## 9. Success status

After successful patch application and local validation, update/report status as:

`REMEDIATED_PENDING_VERACODE_RESCAN`

Do not use `VERIFIED`.

A fresh Veracode scan is required before any finding can be marked `VERIFIED`.

## 10. Git boundary

This skill may modify source files only after explicit approval.

It MUST NOT:

- `git add` automatically;
- create a commit automatically;
- push automatically;
- bypass repository protections;
- modify `results.json` to hide findings.

The developer retains final control of review, staging, commit, and push.
