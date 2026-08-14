---
name: enterprise-main-skill
description: Routes enterprise commands to the appropriate sub-skill.
---

# Enterprise Main Skill

This skill acts only as a command router. It supports Veracode scan and controlled patch-application shortcuts.

It must not perform Veracode scanning, repository analysis, remediation, or terminal monitoring itself.

## Veracode Scan Command

When the user's command is exactly:

`Veracode-S`

or begins with:

`Veracode-S `

immediately delegate execution to:

`.github/skills/veracode-agent/SKILL.md`

## Mandatory Routing Rules

When `Veracode-S` is detected:

1. Do not inspect the repository.
2. Do not scan source files.
3. Do not inspect `.GitHub/output`.
4. Do not read previous Veracode scan results.
5. Do not perform remediation.
6. Do not explain the Veracode workflow before execution.
7. Do not continue the previous Veracode execution session.
8. Load `.github/skills/veracode-agent/SKILL.md`.
9. Follow the Veracode sub-skill instructions exactly.
10. Pass any text after `Veracode-S` to the Veracode sub-skill as the requested target or scan instruction.

## Examples

### Example 1

User:

`Veracode-S`

Action:

Load:

`.github/skills/veracode-agent/SKILL.md`

Then execute the Veracode scan workflow defined by that skill.

### Example 2

User:

`Veracode-S customer-service`

Action:

Load:

`.github/skills/veracode-agent/SKILL.md`

Pass:

`customer-service`

as the requested project/module target.

### Example 3

User:

`Veracode-S payment-service`

Action:

Load:

`.github/skills/veracode-agent/SKILL.md`

Pass:

`payment-service`

to the Veracode sub-skill.


## Veracode Patch Apply Command

When the user's command is exactly:

`Veracode-P`

or begins with:

`Veracode-P `

immediately delegate execution to:

`.github/skills/veracode-patch-apply/SKILL.md`

### Mandatory Patch-Apply Routing Rules

When `Veracode-P` is detected:

1. Do not run a new Veracode scan.
2. Do not regenerate remediation patches.
3. Load `.github/skills/veracode-patch-apply/SKILL.md`.
4. Continue execution according to the patch-apply sub-skill.
5. Pass any text after `Veracode-P` as the requested project/module target.
6. Require the patch-apply sub-skill's explicit developer approval before modifying source files.
7. Do not stage, commit, or push changes automatically.
8. After successful patch application and local validation, leave the result as `REMEDIATED_PENDING_VERACODE_RESCAN`.

### Veracode-P Example

User:

`Veracode-P`

Action:

Load:

`.github/skills/veracode-patch-apply/SKILL.md`

Then execute the controlled patch-application workflow defined by that skill.

User may also specify a target:

`Veracode-P customer-service`

Pass `customer-service` to the patch-apply sub-skill as the requested project/module.

## Important

The main skill is only responsible for routing.

After handing execution to:

`.github/skills/veracode-agent/SKILL.md`

the Veracode sub-skill owns the workflow.

Do not duplicate Veracode execution logic in this file.
