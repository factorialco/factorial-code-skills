---
name: fcode-core-concepts
description: Factorial Code platform architecture and core concepts — processes, modules, execution context, team variables and their inheritance from parent workspaces, datastore, file storage, workspace structure, and naming conventions. Use when building, editing, or reasoning about any Factorial Code (fcode) process, module, or workspace; start here before writing process or module code.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code core concepts

Factorial Code (fcode) is an enterprise integration and automation platform.
You write **processes** (and reusable **modules**) in JavaScript or Python; the
platform handles sandboxing, dependencies, secrets, scheduling, and execution.

This skill is the mental model. For writing actual code, also use
`fcode-javascript` or `fcode-python`; for the CLI workflow, `fcode-cli`.

## Gotchas

These defy reasonable assumptions — get them wrong and the process breaks:

- **Datastore stores only strings and numbers.** Serialize objects with
  `JSON.stringify` / `json.dumps` before `set`, and parse on `get`.
- **Module files are named after their slug, not `index`/`main`.** A module
  lives at `modules/<slug>/<slug>.js` (or `.py`) — e.g.
  `modules/shopify-client/shopify-client.js`. **Never** `modules/<slug>/index.js`
  or `main.py` (those names are reserved for *process* entry files), and never
  put a module file directly under `modules/` without its own folder.
- **Never overwrite the whole `variables.env`.** Read it first and append/patch
  only the specific variable(s); rewriting the file drops every variable not in
  the new content and can break other processes.
- **Never edit `variables.inherited.env`.** It is regenerated on every pull and
  owned by the parent workspace. To change an inherited value for this
  workspace, add the key to `variables.env` — that overrides it.
- **Never hardcode or log secrets.** Use variables/env vars; mask or omit
  secrets from logs.
- **Runtimes are pinned:** JavaScript = **Node.js v22**, Python = **3.13**.

## Key concepts

| Concept | What it is | Key point |
|---|---|---|
| **Process** | The unit of execution (business logic) | Defines input parameters, returns structured results |
| **Module** | Reusable code library | Shared across processes, can be versioned |
| **Workspace version** | One tag (e.g. `v1.0.0`) published on every process and module the team owns | Published from the web UI; retrying the same tag is safe |
| **Version alias** | Movable pointer to a version | `stable` always exists — pin consumers to it; moving it is rollout/rollback |
| **Variables** | Configuration & secrets | Env vars; inherited from parent workspaces; never hardcode secrets |
| **Datastore** | Persistent key-value store | **Strings and numbers only** |
| **Storage** | File storage | Binary files, documents, large payloads |
| **Email** | Built-in transactional email | `fcode.sendMail` / `send_mail`; no SMTP setup, credentials live in the manager |

### Processes

The basic unit of execution, in JavaScript (`index.js`) or Python (`main.py`).
Processes may declare input parameters via JSON Schema (`parametersSchema.json`,
see `fcode-json-schema`) and read them through `fcode.context.parameters`.
Return structured JSON; for webhook-style responses return
`{ status, headers, body }`.

### Modules

Reusable libraries shared across processes — use them to avoid duplication,
encapsulate API clients/integrations, keep process code small, and support
versioning. See the module-naming gotcha above.

### Execution context

Each process runs isolated, with access to:

- **Input parameters**: `fcode.context.parameters`
- **Environment variables**: `process.env.*` / `os.getenv(...)` (or `fcode.env.*`)
- **Execution metadata**: `fcode.execution.*`
- **Request data** (webhooks): request body/headers when applicable

### Variables (configuration & secrets)

Store base URLs, timeouts, API keys, and tokens as variables (never hardcode).
`variables.env` holds team variables (`KEY=VALUE`); `variables.local.env` holds
local-only overrides. See the overwrite gotcha above.

`variables.meta.json` marks each variable's `isSensitive` flag. Sensitive
values never leave the cloud — locally they appear as a `********` placeholder
in `variables.env`; put real values in `variables.local.env`. Details in
`fcode-cli`.

When a secret's real value is needed for local runs, ask the user for it — or
have them put it in `variables.local.env` themselves if they prefer not to
share it. `FACTORIAL_TOKEN` comes from the OAuth flow in the Factorial Code app
details page and is only needed locally (auto-populated remotely); procedure in
`fcode-cli`.

A team variable is also how a webhook is protected: the process names the
variable holding the token it expects, and callers send that value as
`Authorization: Bearer <token>`. Only the variable *name* is stored with the
process, so the token stays out of exports and out of committed files.

Read them at runtime via `fcode.env.*`. To create/update/delete them
programmatically from a process, use the `fcode.variables` helper
(`set`/`get`/`list`/`delete`) — scoped to your team, no API token needed. See
`fcode-javascript` / `fcode-python`.

#### Variables are inherited from parent workspaces

A workspace uses the variables defined in any of its `parentTeamSlugs` parents
without redefining them, alongside processes and modules. Inheritance follows the
same rule those already use: the workspace first, then its **direct** parents
sorted by slug, capped at 5 — so it is **not transitive** (a grandparent's
variables don't reach a grandchild).

- **Don't re-create a parent's variables in a child.** They already resolve
  there. This is why a `deploy-{deployId}` workspace carries only the values
  specific to that customer, while shared defaults and credentials stay in
  `prod-{appId}` / `base-app`.
- **Defining the same key in the child overrides the inherited one** for that
  workspace — the child's value is what its executions see, and the parent's
  entry disappears from the child's list entirely (one entry per key, never two).
  Deleting the override brings the parent's value back.
- **Inherited variables are read-only where they're inherited.** Edit or delete
  them in the workspace that owns them, or override them locally. In the web UI
  they carry an inherited badge and offer **Override here**; the CLI and SDK
  behaviours are in `fcode-cli` and `fcode-javascript` / `fcode-python`.
- **Secrets inherit too, and their real values reach the sandbox.** A process in
  a child workspace reads a parent's secret at execution time. They stay masked
  everywhere else (`null` over GraphQL, `******` over REST, `********` in the
  CLI's inherited file), so process code is the only place a value is readable.
  Treat code in a child workspace as trusted with its parents' credentials.
- **Local runs get the placeholder, not the secret** — put real values in
  `variables.local.env` (see `fcode-cli`).

### Schedules

Run a process on a cron or one-off date/time. Manage schedules from process code
with the `fcode.schedule` helper (`create`/`list`/`get`/`update`/`pause`/
`resume`/`delete`/`deleteForProcess`) — same out-of-the-box auth as the other
helpers. See `fcode-javascript` / `fcode-python`.

### Datastore vs Storage

- **Datastore** — persistent key-value state across runs (last-run timestamps,
  cursors, dedup IDs, small caches). Strings/numbers only.
- **Storage** — files that don't belong in datastore (reports, exports, images,
  PDFs, data extracts).

### Sending email

Send email with the built-in `fcode.sendMail` (`fcode.send_mail` in Python) —
pre-authenticated, no SMTP configuration. The mail server and credentials live in
the executor manager, never in your process. Each execution can send up to 3
emails by default. See `fcode-javascript` / `fcode-python` for usage.

### Versioning & aliases

Processes and modules can be versioned individually, and a **workspace version**
publishes one tag (e.g. `v1.0.0`) on **every process and module the team owns**
at once — resources inherited through `parentTeamSlugs` are never touched. Each
entity's outcome (created / skipped / failed) is recorded in the version's
**manifest**, so re-creating the same tag after a partial failure only publishes
what is still missing.

When a workspace version is published, bare `fcode.import("mod")` /
`fcode.import_module("mod")` calls of workspace-owned modules are pinned to the
tag **inside the published snapshots only** — the working copy is never
modified, and imports that already carry a tag or alias are left untouched.

A **version alias** is a movable pointer to a version. The `stable` alias
always exists and points at the workspace's stable version. Webhooks, forms,
schedules, and module imports accept an alias wherever they accept a tag, so
moving `stable` to another version re-points the whole workspace in one
operation — **rollout and rollback are a single alias change**. Always pin
consumers (webhook URLs, form embeds) to `stable`; how in `fcode-cli` and
`fcode-forms`.

Two consequences of that model:

- **`fcode push` never affects consumers pinned to `stable`.** Pushing updates
  the current (unversioned) code; pinned consumers keep running the released
  version until the alias moves.
- **Deleting a workspace version cascades** — every owned process/module
  version carrying the tag is deleted, together with the aliases, executions,
  and schedules referencing them.

Versions are published and aliases linked from the web UI (team settings →
**Versions** tab). The CLI equivalents (`fcode team:versions:*` /
`team:aliases:*`) are documented in `fcode-cli` — don't create versions or move
`stable` unless explicitly asked.

A **version tag** (`v1.0.0`) is unrelated to `metadata.json` `tags` — those are
process labels (used e.g. for MCP-tool exposure, see `fcode-agent`).

## Decision guidelines

**Module vs inline code**

| Scenario | Recommendation |
|---|---|
| API client used by multiple processes | Create a module |
| Utility helpers used 2+ times | Create a module |
| One-off transformation / single-use logic | Keep inline |

**Datastore vs Variables vs Storage**

| Need | Use |
|---|---|
| Config that rarely changes; secrets/credentials | **Variables** |
| State that changes between runs; cached API responses | **Datastore** |
| Binary files / large exports | **Storage** |

## Naming conventions

| Resource | Convention | Example |
|---|---|---|
| Process slug | kebab-case | `order-sync-shopify` |
| Module slug | kebab-case | `shopify-client` |
| Variables | SCREAMING_SNAKE_CASE | `SHOPIFY_API_KEY` |
| JavaScript functions | camelCase | `fetchOrders()` |
| Python functions | snake_case | `fetch_orders()` |

## Workspace structure (CLI)

A local workspace managed by the `fcode` CLI (see `fcode-cli`):

```
📦 <workspace-name>
┣ 📂 dependencies          # shared deps: package.json (JS) / requirements.txt (Py)
┣ 📂 modules
┃ ┗ 📂 <module-slug>       # one folder per module
┃   ┗ 📜 <module-slug>.js  #   entry file named after the slug (NOT index.js)
┣ 📂 processes
┃ ┗ 📂 <process-slug>      # one folder per process
┃   ┣ 📜 index.js          #   or main.py — the process entry file
┃   ┣ 📜 parametersSchema.json   # input parameter schema (the form)
┃   ┣ 📜 parameters.json   #   default test parameters for `fcode run`
┃   ┣ 📜 metadata.json     #   name, description, tags, webhook/form settings + auth
┃   ┣ 📜 README.md
┃   ┗ 📜 package.json      #   optional process-scoped dependencies
┣ 📜 datastore.json
┣ 📜 team.json             # team settings: inheritance, timezone, error handler, webhook auth
┣ 📜 variables.env         # team variables this workspace owns (KEY=VALUE)
┣ 📜 variables.inherited.env  # variables from parent workspaces (read-only, gitignored)
┣ 📜 variables.local.env   # local overrides (not shared)
┣ 📜 variables.meta.json   # per-variable isSensitive flags
┗ 📂 .fcode
```

Processes and modules also carry `versions/<tag>/` subfolders (e.g.
`versions/v1.0.0/`) holding their published version snapshots — see the
versioning section above. `dependencies/package.json` holds only the inner
`dependencies` object (e.g. `{ "axios": "^1.6.0" }`).

`metadata.json` is where a process's webhook trigger and form settings
(`enabled`, `authMode`, and a marketplace `appRole`) are configured — edit it and
`fcode push`. A webhook is public (`authMode: NONE`), inherits the workspace
`webhookAuth` from `team.json` (`TEAM`), or carries its own header and team
variable (`CUSTOM`). Full field reference in `fcode-cli`.

## General rules

- Validate inputs early — check required parameters and types at the start.
- Handle errors explicitly; throw meaningful, actionable errors, and log every
  caught error with context (what operation, which inputs) before re-throwing.
- Use timeouts/retries for external calls; mind rate limits.
- Log generously through the shared `fcode-logs` module — level-gated logging via
  the `LOG_LEVEL` team variable (default `info`). Log start/end, major decisions,
  and external calls at `info`, and detail (payloads, intermediate state) at
  `debug` (gated off in production). Never log secrets. Usage in
  `fcode-javascript` / `fcode-python`.
- Keep outputs structured (JSON that's easy to consume and debug).
