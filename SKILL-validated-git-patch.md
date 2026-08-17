---
name: veracode-remediation
description: Select a project/module from .GitHub/output/veracode, automatically remediate Critical/Very High and High findings from its latest results.json, ask before including Medium or Low findings, and generate one consolidated patch per affected source file.
---

# Veracode Remediation Skill

## 1. Source of truth

The sole authoritative vulnerability input is a raw Pipeline Scan file named
`results.json` under `.GitHub/output/veracode`.

Do not substitute CSV files, `veracode-findings.json`, manually typed summaries,
or findings from another scan when `results.json` exists.

Do not run a new vulnerability discovery scan. The remediation scope is strictly
limited to findings already present in the selected `results.json`.

## 2. Discover valid projects/modules

Inspect immediate child directories of:

`.GitHub/output/veracode/`

A valid project/module contains at least one `results.json` in either layout:

```text
.GitHub/output/veracode/<project>/<YYYY-MM-DD>/results.json
.GitHub/output/veracode/<project>/<YYYY-MM-DD>/<HHMMSS>/results.json
```

Rules:

- `<project>` is always the directory immediately below `.GitHub/output/veracode`.
- Ignore folders that contain no supported `results.json`.

## 3. Select the project/module

- Use a valid project/module explicitly named by the user.
- If exactly one valid project/module exists, select it automatically and state it.
- If two or more valid projects/modules exist, show a numbered list and ask exactly:

  `Select the project/module to remediate by number or name:`

Wait for the selection before changing source files.

Do not ask the user to select a scan date.

## 4. Select the latest scan automatically

For the selected project, inspect the project layout and choose the
newest scan by:

1. newest valid `YYYY-MM-DD` date directory;
2. newest timestamp/run subdirectory on that date;
3. newest `results.json` modification time when still tied.

Prefer a sibling `scan-status.json` showing a successful/completed scan. Skip any
scan folder without `results.json`.

State the exact repository-relative selected `results.json` path before editing.
Do not combine findings from different scans.


## 5. Severity selection policy

Normalize severity before selecting remediation scope. Use the severity text and
numeric rank available in `results.json`. Treat all of the following as the
highest severity group:

- `Critical`;
- `Very High`;
- numeric severity `5`;
- equivalent highest-severity labels returned by Veracode.

Default remediation scope:

- automatically include every Critical/Very High finding;
- automatically include every High finding;
- do not require confirmation for Critical/Very High or High findings.

Optional remediation scope:

- Medium findings require explicit user approval;
- Low and Very Low findings require explicit user approval;
- Informational findings are excluded unless the user explicitly names them in
  the original request.

After parsing the selected `results.json`, count findings by normalized severity.
When at least one Medium or Low/Very Low finding exists, ask exactly once:

`Critical and High findings will be remediated automatically. Include additional findings? [1] No [2] Medium only [3] Low only [4] Medium and Low:`

Interpret the answer as follows:

1. `No` — remediate only Critical/Very High and High.
2. `Medium only` — also include Medium.
3. `Low only` — also include Low and Very Low.
4. `Medium and Low` — include Medium, Low, and Very Low.

Also accept equivalent natural-language answers. If the response is unclear, ask
again and do not edit source files. If no Medium or Low/Very Low findings exist,
do not ask this question.

Record the severity counts, default-selected findings, user choice, included
finding IDs, and excluded finding IDs in `selected-scan.json` and
`remediation-report.md`. Excluded findings must not be marked remediated or added
to patch metadata.

## 6. Fast parsing and grouping

Parse `results.json` exactly once whenever possible.

For each finding, capture every available field, including:

- finding/flaw/issue ID;
- CWE;
- severity;
- title/category;
- module;
- source file and line;
- method/function/procedure;
- untrusted source;
- risky sink;
- description and remediation guidance;
- data path/call stack;
- status.

Never fabricate missing fields. Use `Not present in results.json` where needed.

After applying the severity selection policy, normalize the repository-relative
source path and build this logical map from selected findings only before opening
source files:

```text
<source-file-1> -> [finding A, finding B, finding C]
<source-file-2> -> [finding D]
```

Performance requirements:

- Do not search the entire repository separately for every finding.
- Do not repeatedly reopen the same source file.
- Open each affected source file once whenever possible.
- Process all findings mapped to that file in one remediation pass.
- Inspect a directly related caller, copybook, include, service, or configuration
  only when it is necessary to understand the reported flow.
- Never scan unrelated files to look for additional vulnerabilities.

## 7. File-by-file remediation

For each affected source file:

1. List all finding IDs, CWEs, severities, lines, procedures/methods, sources, and
   sinks reported for that file.
2. Open the source file once.
3. Correlate overlapping or related findings so one safe correction can address
   multiple findings where appropriate.
4. Trace only the minimum relevant flow for those findings:
   `untrusted source -> transformations -> validation/encoding -> sink`.
5. Apply the smallest technically correct set of changes for all findings in the
   file.
6. Preserve APIs, business behavior, database semantics, authentication,
   authorization, error behavior, record layouts, and logging intent.
7. Validate the consolidated file change once, or validate the narrowest affected
   module once when file-only validation is unavailable.
8. Record any finding that cannot be safely remediated and explain why.

Do not create independent, potentially conflicting edits for each finding in the
same file. Produce one coherent final file diff.

## 8. Preferred remediation patterns

- **CWE-79/80 XSS:** framework templating and context-aware output encoding;
  never bypass Angular sanitization.
- **CWE-89 SQL injection:** prepared statements, bind variables, named
  parameters, and explicit allowlists for dynamic identifiers.
- **CWE-113 CRLF/response splitting:** reject CR/LF and use framework-safe header
  APIs or strict allowlists.
- **CWE-117 log forging:** parameterized/structured logging and centralized
  CR/LF/control-character neutralization without removing security logs.
- **CWE-22 path traversal:** trusted base, resolve, normalize, and verify the
  final path remains inside the base.
- **CWE-78 command injection:** avoid shells; fixed executable plus separated,
  allowlisted arguments.
- **CWE-601 open redirect:** relative destinations or explicit allowlists.

Avoid blanket stripping, generic regex sanitizers, catch-and-ignore, hard-coded
safe values, security-control removal, or broad unrelated refactoring.

## 9. Generate one validated Git patch per source file

Derive `<scan-date>` from the selected scan's date directory, not from today's
clock date.

Create:

```text
.GitHub/remediation/veracode/<project-or-module>/<scan-date>/
```

Generate exactly one patch for each changed source file. The patch filename must
be based on the source file's basename, preserving the original extension,
followed by `.patch`.

Examples:

```text
CUAF610.cbl         -> CUAF610.cbl.patch
CustomerDao.java   -> CustomerDao.java.patch
login.component.ts -> login.component.ts.patch
```

When two changed files in different directories have the same basename, avoid a
collision by adding a stable parent-directory qualifier:

```text
accounts-CUAF610.cbl.patch
payments-CUAF610.cbl.patch
```

Do not generate `finding-<id>.patch` files.

### 9.1 Mandatory Git-generated patch rule

The remediation agent MUST NOT manually construct unified-diff syntax.

Do not manually write or calculate:

```text
--- a/<path>
+++ b/<path>
@@ -oldStart,oldCount +newStart,newCount @@
```

Do not calculate hunk line counts yourself. Do not concatenate independently
generated patch fragments. Do not repair an invalid patch by manually changing
`@@` line numbers.

The final patch body MUST be produced by Git from an original file and a
remediated file, or from the repository working-tree change.

Preferred repository flow when the target file is tracked by Git:

1. Preserve the original source content.
2. Apply all selected remediations for that source file as one coherent source
   change.
3. Ask Git to generate the diff for only that source file:

   ```bash
   git diff --binary --no-ext-diff -- <repository-relative-source-path>
   ```

4. Capture Git's output exactly as the patch body.
5. Do not rewrite the `diff --git`, `index`, `---`, `+++`, or `@@` lines.

Alternative isolated flow when the source file cannot safely be changed in the
working tree:

1. Create temporary `original` and `remediated` copies outside the remediation
   output directory.
2. Keep `original` byte-for-byte equivalent to the source file before changes.
3. Apply all selected fixes to `remediated`.
4. Generate the diff with Git:

   ```bash
   git diff --no-index --binary --no-ext-diff -- <original-file> <remediated-file>
   ```

5. Normalize only the file path labels when necessary to make them
   repository-relative. Do not change hunk headers or hunk content.
6. Remove temporary files after successful patch generation and validation.

### 9.2 Patch metadata must not corrupt the patch

Do NOT prepend `#` comments or prose before the Git diff in the `.patch` file.
A patch intended for `git apply` must begin with Git-generated patch content,
normally:

```text
diff --git a/<path> b/<path>
```

Store finding metadata in `remediation-report.md` and `selected-scan.json`
instead of inserting metadata comments before the unified diff.

For each generated patch, record in those metadata files:

- source file;
- patch filename;
- included finding IDs;
- CWEs;
- severity;
- remediation status;
- skipped/failed finding IDs and reasons;
- patch validation command and result.

### 9.3 Mandatory patch validation gate

Every generated patch MUST pass Git validation before it is published as a
successful remediation artifact.

Run from the repository root:

```bash
git apply --check --whitespace=nowarn <patch-file>
```

Validation rules:

- Exit code `0` means the patch syntax and target application are valid.
- If validation fails with `corrupt patch at line ...`, `patch fragment without
  header`, `malformed patch`, hunk errors, or any other error, the patch is
  invalid.
- Never report an invalid patch as generated successfully.
- Never leave an invalid `.patch` file in the final remediation output as if it
  were usable.
- Regenerate the patch from the preserved original source and the coherent
  remediated version by using Git again.
- Never manually edit hunk headers to make validation pass.
- Re-run `git apply --check --whitespace=nowarn` after regeneration.
- If validation still fails, mark the file remediation `FAILED_PATCH_VALIDATION`,
  record the exact Git error in `remediation-report.md`, and do not claim the
  finding is remediated.

When validation succeeds, record:

```text
PATCH_STATUS: VALID
```

When validation fails after regeneration, record:

```text
PATCH_STATUS: INVALID
```

### 9.4 One coherent patch per source file

All compatible selected findings for the same source file MUST be fixed in one
coherent remediated file state before Git generates the patch.

Never:

- create one diff fragment per finding and concatenate them;
- create multiple independently calculated hunks and merge them manually;
- generate a patch from already partially remediated content without preserving
  the original source baseline.

This ensures that a file such as `CUAF610.cbl` with several Veracode findings
produces one Git-generated patch such as:

```text
CUAF610.cbl.patch
```

containing every compatible change for that source file.

### 9.5 Patch content requirements

- Use repository-relative paths in the final Git patch.
- Include all compatible changes for that source file in the same patch.
- Do not include credentials, tokens, binaries, generated build output, or
  Veracode result files.
- Keep output deterministic.
- Preserve source-file line endings where practical.
- Do not add explanatory prose inside the Git-generated diff.

Also write:

```text
remediation-report.md
selected-scan.json
```

`selected-scan.json` must record:

- selected project/module;
- application type when present;
- selected repository-relative `results.json` path;
- selected scan date;
- timestamp/run folder when present;
- remediation start time;
- total findings and counts by normalized severity;
- automatically selected Critical/Very High and High finding IDs;
- the user's Medium/Low choice;
- included and excluded finding IDs;
- affected source files;
- mapping of each generated patch filename to its source file, finding IDs, CWEs,
  and patch validation status.

For every report entry include finding evidence, root cause, remediation decision,
changed file, consolidated patch filename, validation command, validation result,
and status:

`REMEDIATED_PENDING_VERACODE_RESCAN`

Only use that remediation status when the source correction is complete and the
patch passed `git apply --check`.

A failed patch validation must instead be reported as:

`FAILED_PATCH_VALIDATION`

Only a new Veracode scan can establish `VERIFIED`.
## 10. Validation strategy

Patch validation is mandatory and separate from source/build validation.

For every generated patch, first run from repository root:

```bash
git apply --check --whitespace=nowarn <patch-file>
```

Only after the patch passes this check should normal source/build validation be
reported as successful.

To reduce execution time:

- validate every patch exactly once after its final generation, except when a
  failed patch is regenerated, in which case validate the regenerated patch again;
- validate once per changed file when a compiler/linter supports it;
- otherwise validate once per affected module after all file changes are prepared;
- do not run the same build or test repeatedly for individual findings in one file;
- do not run a full repository build unless required by the project and no narrower
  validation exists;
- record commands, results, and limitations accurately.

## 11. Completion summary

Report:

- selected project/module;
- selected `results.json` path and scan date;
- total findings by severity, selected findings, and excluded findings;
- one row per changed source file showing its consolidated patch and finding IDs;
- remediated, skipped, and failed counts;
- Git patch validation result for every generated patch;
- source/build validation performed;
- remediation output directory;
- requirement to run a fresh Veracode scan.
