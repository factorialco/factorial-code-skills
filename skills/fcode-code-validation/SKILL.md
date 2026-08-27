---
name: fcode-code-validation
description: Pre-production review of a Factorial Code app — clone the workspace read-only, run the static check catalog, classify findings as Blocker/Warning/Suggestion, and write APP_VALIDATION_REPORT.md with a ✅/❌ promotion verdict. Use when validating, reviewing, or auditing a Factorial Code (fcode) app or workspace before a release, or when asked whether an app is ready to ship.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — App validation

A pre-flight for the platform's own release validation (its criteria are in
`fcode-ama` references/journey.md §8), run locally before the release is
requested, adding the review dimensions a human gatekeeper would: forms and
lifecycle, i18n, logging hygiene, security.

It serves two audiences with the same procedure: an app developer
self-checking before requesting a release, and a platform reviewer gating
one. Either way the input is a workspace ("team") slug and the output is
`APP_VALIDATION_REPORT.md` in the cloned workspace plus a ✅/❌ verdict in
chat.

This skill owns only the procedure, the check catalog, the severities, and
the report format. Every *rule* it checks is owned by another skill and
cited on the catalog row — read the owner when a finding needs context.

## Gotchas

- **This review is read-only.** The only permitted writes are the clone
  itself and `APP_VALIDATION_REPORT.md`. Never `fcode push`, never
  `--force`, never `settings:versions:create` / `settings:aliases:set`, never edit
  workspace files, never `fcode add` the report.
- **Never `fcode run` during the review.** Processes may call real APIs
  with credentials inherited from parent workspaces
  (`fcode-core-concepts`). This is a *static* review of code and config.
- **The clone slug is an encoded `dev-…` token, not the app's UUID** from
  the browser URL — copy it from the app's "How to build locally" guide on
  its Development tab (see `fcode-cli`). A UUID-shaped slug will not clone.
- **Any Blocker → ❌.** Warnings and Suggestions never flip the verdict.
- **Ambiguity blocks the review.** If the user names a development team
  with several apps, or it's unclear which workspace (`dev-` / `deploy-` /
  `prod-`) to review, list the candidates and ask before cloning. `dev-` is
  the normal review target.
- **`********` placeholders in pulled variables are normal** — a masked
  secret is never a finding, and the review never needs real secret values.
  Never ask for them.
- **A failing `fcode clone`** usually means UUID confusion or missing
  access — point at the access flow in `fcode-ama`; don't retry blindly.
- **A mixed JS/Python workspace is fine** — review each process with its
  own language's rules and note the mix in the report header.
- **Review only what the workspace owns.** Inherited resources — modules,
  processes, and variables from `parentTeamSlugs` parents,
  `variables.inherited.env`, `i18n/<locale>.inherited.yaml` — were already
  validated when their owning workspace was published. Don't audit their
  content; findings may only concern how *this* workspace uses them.

## Severity model

| Severity | Meaning | Effect on verdict |
|---|---|---|
| Blocker | Breaks at runtime, mishandles credentials, or fails a criterion the platform release gate checks | Any one → ❌ |
| Warning | Violates a documented rule; expect reviewer pushback or future breakage | Advisory |
| Suggestion | Improvement aligned with platform conventions | Advisory |

## Procedure

1. **Resolve the workspace.** Confirm the slug is the encoded token (see
   Gotchas) and which workspace kind it is; disambiguate with the user when
   needed. Record the kind in the report header.
2. **Acquire.** Missing CLI: ask the user, then
   `pnpm install -g @factorialco/fcode-cli` — the one permitted install.
   Fresh copy: `fcode clone <workspace-slug> --skipSkillsSetup`. Already
   cloned: `fcode pull`; on divergence never `--force` — ask whether to
   review the local state as-is and record the answer in the report header.
3. **Inventory** (read-only). Detect the language per process (`index.js`
   vs `main.py`). List processes, modules, `appRole`s, and webhook/form
   settings from each `processes/<slug>/metadata.json`; read `settings.json`,
   the three variables files, `variables.meta.json`, and `i18n/`. Run
   `fcode status`, `fcode variables:status`, `fcode settings:status`, and
   `fcode i18n:status`. Separate owned from inherited as you go — the
   catalog applies to owned resources only (see Gotchas).
4. **Run the check catalog.** Read `references/checks.md` and evaluate
   every check against the inventory, recording `file:line` evidence.
   Whether step 2 was a fresh clone or a pre-existing local workspace
   decides if the STRUCT layout checks run fully or are marked
   "N/A — fresh clone".
5. **Classify.** Apply each catalog row's severity; deviate only where the
   row allows it, and say why in the finding.
6. **Write the report** to `<workspace>/APP_VALIDATION_REPORT.md` following
   `references/report-template.md`, overwriting any previous run.
7. **Print the verdict in chat**: the ✅/❌ line, per-severity counts, and
   the Blocker titles.

## Check categories

`STRUCT` · `LANG` · `FORM` · `I18N` · `SEC` · `REUSE` · `LOG` · `LIFE` ·
`REL` — the full catalog, with per-check severities and owner citations, is
in `references/checks.md`.
