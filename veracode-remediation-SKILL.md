---
name: veracode-remediation
description: Generate reviewable Veracode remediation patches without modifying application source files. Critical/Very High and High are included by default; Medium and Low require approval.
---

# Veracode Remediation Skill

## 1. Purpose

This skill generates remediation patch files only.

Patch generation and patch application are deliberately separated for enterprise safety and developer control.

The source tree MUST remain unchanged while this skill is running.

## 2. Source of truth

Use the selected raw Veracode `results.json` under:

`.GitHub/output/veracode/<project-or-module>/.../results.json`

Do not invent findings and do not substitute unrelated scan output.

## 3. Project/module selection

Discover valid immediate child directories under:

`.GitHub/output/veracode/`

A valid project/module contains at least one supported `results.json`.

If multiple projects/modules exist, ask:

`Select the project/module to remediate by number or name:`

Automatically choose the latest valid scan for the selected project/module.

## 4. Severity policy

Automatically include:

- Critical;
- Very High;
- High;
- numeric severity 5 or equivalent highest-severity labels.

When Medium or Low/Very Low findings exist, ask exactly once:

`Critical and High findings will be remediated automatically. Include additional findings? [1] No [2] Medium only [3] Low only [4] Medium and Low:`

Do not include unapproved Medium/Low findings.

## 5. Finding grouping

Parse `results.json` once whenever possible and map selected findings by normalized repository-relative source file:

```text
<source-file-1> -> [finding A, finding B]
<source-file-2> -> [finding C]
```

Read each affected source file only as needed. Do not perform unrelated repository-wide scanning.

## 6. Remediation preparation

For each affected source file:

1. correlate all selected findings for that file;
2. determine the smallest safe correction;
3. preserve application behavior and security controls;
4. prepare the corrected version in memory/workspace only;
5. compare original content with the proposed corrected content;
6. generate one consolidated unified-diff patch.

### Absolute source-protection rule

During this skill, NEVER:

- overwrite the original source file;
- use `git apply`;
- use the OS `patch` command;
- invoke an IDE patch-apply operation;
- stage source changes with Git;
- commit source changes;
- silently edit application files as part of remediation generation.

If patch generation tooling would modify the source file first, do not use that approach. Generate the diff from original content and an in-memory or temporary proposed version.

## 7. Output directory

Use the selected scan date:

```text
.GitHub/remediation/veracode/<project-or-module>/<scan-date>/
```

Generate exactly one `.patch` file per affected source file.

Examples:

```text
CustomerDao.java.patch
CustomerController.java.patch
CUAF610.cbl.patch
login.component.ts.patch
```

Also generate:

```text
selected-scan.json
remediation-report.md
```

## 8. Patch metadata

At the top of each patch include metadata comments such as:

```text
# Source file: customer-service/src/main/java/.../CustomerDao.java
# Finding IDs: 1201, 1202
# CWEs: CWE-89, CWE-117
# Status: PATCH_GENERATED_PENDING_DEVELOPER_APPROVAL
--- a/customer-service/src/main/java/.../CustomerDao.java
+++ b/customer-service/src/main/java/.../CustomerDao.java
```

Use repository-relative paths.

## 9. Report status

Use these statuses accurately:

- `PATCH_GENERATED_PENDING_DEVELOPER_APPROVAL` — patch created but not applied;
- `SKIPPED` — finding intentionally not changed;
- `FAILED` — safe patch could not be generated.

Do NOT use `REMEDIATED_PENDING_VERACODE_RESCAN` until the patch has actually been applied and locally validated.

Do NOT use `VERIFIED` until a fresh Veracode scan confirms closure.

## 10. Completion

Stop immediately after patch/report generation and summarize:

- selected project/module;
- selected scan;
- findings included/excluded;
- generated patch files;
- skipped/failed findings;
- output directory;
- confirmation that source files were NOT modified;
- next step: review the patches and explicitly run `Apply Veracode patches` when ready.
