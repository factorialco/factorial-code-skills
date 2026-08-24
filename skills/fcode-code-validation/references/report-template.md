# App validation — report template

Write the report to `APP_VALIDATION_REPORT.md` at the root of the cloned
workspace, overwriting any previous run.

## Authoring rules

- **Fixes are instructions for an agent.** Each fix is an imperative,
  file-scoped, self-contained instruction the developer's coding agent can
  execute without reading this skill or the catalog: name the file, the
  change, and the commands involved. Don't say "see FORM-05" — say what to
  do.
- **Cite the rule's owner** so a human reviewer can verify the rationale:
  one sentence of rule plus the owning skill (and reference file when the
  rule lives there).
- N/A categories appear in the summary with the reason — never silently
  omitted.

## Template

```markdown
# App Validation Report

| | |
|---|---|
| App | <app name from metadata or the user> |
| Workspace | `<workspace-slug>` (<dev/deploy/prod>; <fresh clone / pre-existing, in sync / diverged — reviewed local state>) |
| Language | <JavaScript (Node.js v22) / Python 3.13 / mixed: …> |
| Reviewed | <YYYY-MM-DD> |
| Verdict | ❌ **Not ready for promotion** — <N> Blocker(s) *or* ✅ **Ready for promotion** |

## Summary

| Category | Blockers | Warnings | Suggestions |
|---|---|---|---|
| STRUCT — structure & platform rules | 0 | 1 | 0 |
| LANG — language rules | 0 | 0 | 0 |
| FORM — forms & appRole lifecycle | 1 | 0 | 0 |
| I18N — internationalization | 0 | 1 | 0 |
| SEC — security & secrets | 1 | 0 | 0 |
| REUSE — base-app reuse | 0 | 0 | 1 |
| LOG — logging | 0 | 1 | 0 |
| LIFE — install/uninstall hygiene | N/A — no lifecycle | | |
| REL — release readiness | 0 | 0 | 0 |
| **Total** | **2** | **3** | **1** |

## Blockers

### [SEC-01] Hardcoded Acme API key
- **Where:** `processes/order-sync/index.js:12`,
  `modules/acme-client/acme-client.js:3`
- **Rule:** Never hardcode secrets — use team variables
  (`fcode-core-concepts` §Gotchas).
- **Fix:** Create a sensitive team variable `ACME_API_KEY`
  (`fcode variables:add --sensitive`, then set its value in the cloud), read
  it via `process.env.ACME_API_KEY` at both locations, and delete the
  literal. For local runs put the real value in `variables.local.env`, never
  in `variables.env`.

## Warnings

<same shape as Blockers>

## Suggestions

<same shape as Blockers>

## Checks passed

- STRUCT: 03, 05, 06, 07, 08 (01, 02 N/A — fresh clone)
- LANG: 01–06
- FORM: 01, 03, 04, 06–11
- I18N: 01, 03–06
- SEC: 02–09
- REUSE: 01–05
- LOG: 01, 03
- LIFE: N/A — no lifecycle
- REL: 01–05

## Re-running validation

After applying fixes, run the `fcode-code-validation` skill again with
workspace `<workspace-slug>`; the verdict flips to ✅ only when zero Blockers
remain.
```
