# App validation — check catalog

Every check is a one-line assertion; the full rule lives with the **owner**
skill cited on the row — read it there when a finding needs more context or
the fix isn't obvious. Evaluate every check against the whole workspace and
record `file:line` evidence for each hit. When one check fires in several
places, report it as **one finding with a location list**, not one finding
per site.

**Scope: owned resources only.** Inherited modules, processes, variables
(`variables.inherited.env`), and locales (`i18n/<locale>.inherited.yaml`)
were validated when their owning workspace was published — never audit
their content or report findings inside them. Checks apply to what this
workspace owns, including how its code *uses* inherited resources
(e.g. STRUCT-04's shadowing copies, LIFE-05's no-op deletes).

Severity legend: **B** Blocker (any one → ❌), **W** Warning, **S**
Suggestion. A row's severity is the default; escalate or downgrade only when
the row itself says so, and state the reason in the finding.

## STRUCT — structure & platform rules

Layout checks (01, 02, 06, 07) only apply fully when reviewing a
**pre-existing local workspace** that may hold unpushed changes; a fresh
clone mirrors the cloud state, so on a fresh clone give them a quick sanity
pass and mark the category "N/A — fresh clone" when clean. Code-level checks
(03, 04, 05) always run.

| ID | Check | Sev | Owner |
|---|---|---|---|
| STRUCT-01 | Module entry file is `modules/<slug>/<slug>.js\|py` — never `index.js`/`main.py`, never a file directly under `modules/` | B | `fcode-core-concepts` §Gotchas |
| STRUCT-02 | Process entry file is `index.js` (JS) or `main.py` (Python) inside `processes/<slug>/` | B | `fcode-core-concepts` §Workspace structure |
| STRUCT-03 | Datastore `set` calls store only strings/numbers (objects serialized first) | B | `fcode-core-concepts` §Gotchas |
| STRUCT-04 | No parent variables copied into `variables.env` (they shadow the parent — incl. untouched `********` shadowing a secret); flag, don't delete | W | `fcode-cli` §The three variables files |
| STRUCT-05 | No blank-value keys in `variables.env` (declaring a key overrides the inherited value with `""`) | W | `fcode-cli` §The three variables files |
| STRUCT-06 | Naming conventions: kebab-case slugs, SCREAMING_SNAKE_CASE variables, camelCase (JS) / snake_case (Py) functions | W | `fcode-core-concepts` §Naming conventions |
| STRUCT-07 | `dependencies/package.json` holds only the inner dependencies object; `@add-package` comment present where import name ≠ package name | W | `fcode-core-concepts` §Workspace structure, `fcode-javascript`/`fcode-python` §Dependencies |
| STRUCT-08 | Local temp files written under `TMP_DATA_DIR`, not arbitrary paths | S | `fcode-javascript`/`fcode-python` §Datastore & storage |

## LANG — language rules

Detect the language per process (`index.js` vs `main.py`) and apply that
language's skill; a mixed workspace is fine — review each process with its
own rules and note the mix in the report header.

| ID | Check | Sev | Owner |
|---|---|---|---|
| LANG-01 | JS exports `module.exports = { main }`; Python defines top-level `main()`; neither ever calls `main()` itself | B | `fcode-javascript`/`fcode-python` §Gotchas |
| LANG-02 | `fcode.import(...)` / `fcode.import_module(...)` names are hardcoded string literals, never variables | B | `fcode-javascript`/`fcode-python` §Gotchas |
| LANG-03 | `fcode.i18n` is never aliased — every call is literal (aliasing throws "i18n is disabled" at runtime) | B | `fcode-i18n` §Gotchas |
| LANG-04 | Return values match a supported shape: `{message}`, `{status, headers, body}`, `{transient: true, data}`, `{nextProcessId}`, `{redirect}` | W | `fcode-javascript`/`fcode-python` §Return values |
| LANG-05 | Main flow wrapped in `try/catch` (`try/except`); code doesn't read `fcode.env` expecting a value it `fcode.variables.set` in the same run (snapshot at start) | W | `fcode-javascript`/`fcode-python` §Gotchas, §Variables & schedules |
| LANG-06 | Idiom: `const`/`let` never `var`; `async/await` for async work; PEP 8 + type hints in Python | S | `fcode-javascript`/`fcode-python` §Gotchas |

## FORM — forms & appRole lifecycle

| ID | Check | Sev | Owner |
|---|---|---|---|
| FORM-01 | At most one process per `INSTALL`/`SETTINGS`/`UNINSTALL` appRole (on a clash the first slug alphabetically wins, silently) | B | `fcode-ama` references/troubleshooting.md §Demo companies & Dev Marketplace |
| FORM-02 | A stored secret is never echoed back to a form — pre-render reports only *whether* it is set, blank submit keeps the value; escalation: echoing the value is **B**, marking the secret field `required` (locking users out of partial edits) is **W** | B/W | `fcode-forms` references/advanced.md §Pre-filling current values |
| FORM-03 | `form.authMode` is intentional: a public (`NONE`) form on a process that handles credentials or writes data is **B**; a form whose intended audience is unclear is **W** (omitting the field keeps it protected — that's fine) | B/W | `fcode-forms` §Restrict who can open the form |
| FORM-04 | No secrets in embed code, embed `options`, or the schema — they all reach the browser | B | `fcode-forms` §Gotchas |
| FORM-05 | `SETTINGS` (and re-openable `INSTALL`) forms pre-fill current values via `preRenderProcess` + `#/variables/<name>` refs, reading through `fcode.env`, not an empty form | W | `fcode-forms` references/advanced.md §Pre-filling current values |
| FORM-06 | Work behind a form finishes well inside the ~1-minute request timeout, or goes async (`async: true` / `fcode.processes.run` + quick ack) | W | `fcode-forms` §Keep it fast, or go async |
| FORM-07 | The workspace owns configuration-like variables (credentials, mappings, per-customer settings) **and** an `INSTALL`/`SETTINGS` form exists to set them — fire this only when such variables exist with no form path; a utility app with neither is fine | W | `fcode-forms` §Enable a form, `fcode-cli` §Process metadata |
| FORM-08 | `preRenderProcess` defaults missing state (`?? 60`, `\|\| "{}"`) instead of throwing — a throw makes the form unopenable | W | `fcode-forms` references/advanced.md §Server-side pre-render |
| FORM-09 | Form-uploaded files only needed transiently are deleted at the end of the process | W | `fcode-forms` §Automatic file uploads |
| FORM-10 | Result `message`/`errorMessage` and schema markdown blocks are markdown, not HTML (HTML is dropped, never rendered) | W | `fcode-forms` §Handle submission results |
| FORM-11 | Schema quality: secret inputs use `isSensitive: true`; `nextProcessId` carries a slug, not a UUID; schema follows `fcode-json-schema` | S | `fcode-json-schema`, `fcode-forms` §Multi-step forms |

## I18N — internationalization

Lack of i18n is **never a Blocker** — a private app may legitimately ship in
one language. Only I18N-01 blocks, and it is a reserved-name runtime issue,
not a coverage issue.

| ID | Check | Sev | Owner |
|---|---|---|---|
| I18N-01 | No form field or webhook body field named `locale` (reserved — stripped before parameters are built, like `version_tag` and `async`) | B | `fcode-i18n` §Gotchas |
| I18N-02 | When locales exist: the primary locale covers every key referenced in code and schemas (it is the fallback; a gap ships raw dotted keys to end users) | W | `fcode-i18n` §How the execution locale is chosen |
| I18N-03 | User-visible strings go through `fcode.i18n`: form titles/descriptions/placeholders, result messages, email subjects/bodies, user-visible webhook bodies — **W** when the workspace has locales, **S** when it has none | W/S | `fcode-i18n` §Internationalizing existing code |
| I18N-04 | Dynamic parts use `%{placeholders}` — never concatenated translated fragments; form-token args are a flat object of scalars | W | `fcode-i18n` §Internationalizing existing code, §Gotchas |
| I18N-05 | Logs, developer-facing errors, datastore keys, variable names, and slugs are NOT translated | S | `fcode-i18n` §Internationalizing existing code |
| I18N-06 | Keys named `<process-slug>.<area>.<name>` with shared strings under `common.*` | S | `fcode-i18n` §Internationalizing existing code |

## SEC — security & secrets

| ID | Check | Sev | Owner |
|---|---|---|---|
| SEC-01 | No hardcoded secrets, tokens, or credentials anywhere: code, schemas, `metadata.json`, `parameters.json`, READMEs, committed files | B | `fcode-core-concepts` §Gotchas |
| SEC-02 | No secrets in log calls at any level (incl. payload dumps at `debug` that carry credentials) | B | `fcode-javascript`/`fcode-python` §Logging |
| SEC-03 | No real sensitive value in any synced or committed file — masked `********` entries in `variables.env` are the normal state; real values belong only in `variables.local.env` (never pushed) | B | `fcode-cli` §Getting secret values for local runs, §Variable sensitivity |
| SEC-04 | Credential-looking variables are `isSensitive: true` in `variables.meta.json` (immutable once pushed — the fix is recreating the variable) | B | `fcode-cli` §Variable sensitivity |
| SEC-05 | Webhook auth is correct: `authMode: TEAM` has a `webhookAuth` in `team.json` (per-workspace, not inherited — missing = every call rejected, **B**); an enabled webhook with `authMode: NONE` doing sensitive work is **B**; a bespoke `headerName` without a sender constraint is **S** (loses `Authorization` redaction) | B/S | `fcode-cli` §Process metadata, §Team settings |
| SEC-06 | Sensitive data returned to callers uses `{transient: true, data}` so it is not persisted in execution results | W | `fcode-javascript`/`fcode-python` §Return values |
| SEC-07 | Runtime `fcode.variables.set` flags are right: secrets stay default-sensitive, plain config passes `sensitive: false` — a secret created non-sensitive is **W**, config created sensitive is **S** | W/S | `fcode-javascript`/`fcode-python` §Variables & schedules |
| SEC-08 | No eval-like execution of user-controlled input (`eval`, `Function`, `exec`, dynamic `require` of user data) — child workspaces run this code with the parents' credentials | W | `fcode-core-concepts` §Variables are inherited |
| SEC-09 | OAuth scopes match the API calls actually made — request only what the app needs (static approximation; note uncertainty) | S | `fcode-ama` references/journey.md §4 |
| SEC-10 | Every Factorial webhook subscription the app creates (`setupWebhook` from `factorial-utils`, or a direct API call) carries a challenge token, and the receiving process verifies it from the `x-factorial-wh-challenge` header — via the workspace `webhookAuth` + `authMode: TEAM`, or the base `checkWebhookChallenge()` helper | B | `fcode-cli` §Team settings, `fcode-examples` §The base workspaces |

## REUSE — base-app reuse

The platform release gate explicitly checks "correct use of the Factorial
Code templates (not reinventing what the base workspaces provide)" —
`fcode-ama` references/journey.md §8 — which is why reimplementations of the
core base modules block. The base-module table is in `fcode-examples` §The
base workspaces.

| ID | Check | Sev | Owner |
|---|---|---|---|
| REUSE-01 | Factorial API calls go through the `factorial-sdk` module (`createFactorialClient()`), not a hand-rolled HTTP client against the Factorial API | B | `fcode-examples` §The base workspaces |
| REUSE-02 | Email goes through `fcode.sendMail` / `fcode.send_mail`, never a custom SMTP/mail library (**B**); loops that can exceed the 3-emails-per-execution cap are **W** | B/W | `fcode-core-concepts` §Sending email |
| REUSE-03 | `OutboundSync` subclasses never override `run()` (override `process()` and hooks only) | B | `fcode-examples` references/integration-acme.md §The OutboundSync contract |
| REUSE-04 | Webhook auth relies on platform `authMode`, not token-checking code inside the process | W | `fcode-examples` §Pattern index |
| REUSE-05 | `factorial-utils` helpers (`getCompanyId`, `setupWebhook`, `listWebhookSubscriptions`, `deleteWebhookSubscription`) are imported, not reimplemented | W | `fcode-examples` §The base workspaces |
| REUSE-06 | Where they fit, the `fcode-forms` module (schema builders), `mail-helper` (`brandedHtml`), and `error-handler` are used instead of hand-built equivalents | S | `fcode-examples` §The base workspaces |

## LOG — logging

The user-facing bar is twofold: enough logging to diagnose future errors,
and not so much that production output floods.

| ID | Check | Sev | Owner |
|---|---|---|---|
| LOG-01 | Logging goes through the shared `fcode-logs` module (`LOG_LEVEL`-gated), not bare `console.*` / `print` | W | `fcode-javascript`/`fcode-python` §Logging |
| LOG-02 | Coverage: start/end of the main flow, every external call, and major decisions at `info`; every `catch` logs the error with context (operation, inputs) before re-throwing | W | `fcode-javascript`/`fcode-python` §Logging |
| LOG-03 | No flooding: payload/state dumps sit at `debug`, not `info`; no per-item `info` logs inside large loops (aggregate counts at `info` instead) | S | `fcode-javascript`/`fcode-python` §Logging |

## LIFE — install/uninstall hygiene

Applies to apps with an install lifecycle (any `appRole` process, or code
creating external webhooks/schedules). For a pure utility app, mark the
category "N/A — no lifecycle".

| ID | Check | Sev | Owner |
|---|---|---|---|
| LIFE-01 | Install records every created resource id (webhooks, schedules, vendor objects) in the datastore; an `UNINSTALL` process that cannot find what install created is **B** (silent residue), install not recording ids with no uninstall yet is **W** | B/W | `fcode-examples` references/custom-app-linear.md §Adapting to another vendor |
| LIFE-02 | Uninstall is best-effort (per-step try/catch, reports what it removed) behind an explicit confirm flag | W | `fcode-examples` references/custom-app-linear.md §Uninstall |
| LIFE-03 | An app that creates external webhooks or schedules has an `UNINSTALL` process to tear them down | W | `fcode-examples` references/custom-app-linear.md §Architecture |
| LIFE-04 | Polling uses a datastore cursor plus idempotency (dedup map or vendor upsert) | S | `fcode-examples` references/custom-app-linear.md §Runtime — scheduled poll |
| LIFE-05 | Cleanup code knows `fcode.variables.delete` on an inherited key is a silent no-op (check `resolvingTeamSlug` when reporting removals) | S | `fcode-javascript`/`fcode-python` §Inherited variables |

## REL — release readiness

| ID | Check | Sev | Owner |
|---|---|---|---|
| REL-01 | The workspace has at least one process — an empty workspace has nothing to promote | B | `fcode-core-concepts` §Processes |
| REL-02 | `fcode status` / `team:status` / `variables:status` / `i18n:status` show local and cloud in sync — on divergence, note in the report header which state was reviewed | W | `fcode-cli` §Command flow |
| REL-03 | Consumers found in code/READMEs pin the `stable` alias: `?version_tag=stable` on webhook URLs, `processVersion: "stable"` on embeds | W | `fcode-forms` §Pin the form to a version, `fcode-cli` §Calling a webhook |
| REL-04 | `errorHandlerConfig.processSlug` in `team.json` resolves to an existing process | W | `fcode-cli` §Team settings |
| REL-05 | Slugs are settled (renaming breaks every pasted embed and webhook URL); each locale file is under 256 KB | S | `fcode-forms` §Embed a form, `fcode-i18n` §Locale files |
