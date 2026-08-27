---
name: fcode-core-concepts
description: Factorial Code platform architecture and core concepts — processes, modules, execution context, variables, inheritance from parent workspaces, datastore, file storage, workspace structure, and naming conventions. Use when building, editing, or reasoning about any Factorial Code (fcode) process, module, or workspace; start here before writing process or module code.
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
- **Never edit inherited resources** (`variables.inherited.env`,
  `i18n/<locale>.inherited.yaml`, inherited processes/modules) — they are owned
  by parent workspaces and regenerated on every pull. Override a variable or
  locale key by defining it in this workspace's own file; edit processes and
  modules in the workspace that owns them.
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
| **Locales** | Per-language translation files (`i18n/<locale>.yaml`) | Resolved by `fcode.i18n`; inherited key by key from parents — see `fcode-i18n` |
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

### Inheritance from parent workspaces

A workspace uses the **processes, modules, variables, i18n locales, and
dependencies** of its `parentTeams` parents as if they were its own.
Resolution is the same for all of them: the workspace first, then its
**direct** parents in the order they are configured, capped at 5 — so it is
**not transitive** (a grandparent's resources don't reach a grandchild).

- **Read, call, and import inherited resources freely** — an inherited module
  imports exactly like an owned one, and a child's code can depend on a parent's
  process — but they are **read-only where they're inherited**: never modify,
  delete, or reschedule them, not by editing files, not through the MCP tools.
  Edit them in the workspace that owns them.
- **Variables, locales, and dependencies are overridden by redefining locally**:
  a key in the child's `variables.env`, `i18n/<locale>.yaml`, or
  `dependencies/package.json` / `requirements.txt` wins over the parent's — key
  by key, and package by package for dependencies, where the child's specifier
  *replaces* the parent's because two versions of one package can't be installed
  side by side. Deleting the override brings the parent's back. Processes and
  modules have no override — change them in the owner.
- **Only a parent's *installed* dependencies inherit.** A manifest change the
  parent saved but hasn't installed doesn't reach its children; once installed,
  the child picks it up on its next execution, with nothing to install there.
  A package a parent already provides doesn't need declaring in the child.
- **Don't re-create a parent's variables in a child.** They already resolve
  there. This is why a `deploy-{installationId}` workspace carries only the
  values specific to that customer, while shared defaults and credentials stay
  in `prod-{appId}` / `base-app`. In the web UI inherited variables carry an
  inherited badge and offer **Override here**.
- **Secrets inherit too, and their real values reach the sandbox.** A process in
  a child workspace reads a parent's secret at execution time. They stay masked
  everywhere else (`null` over GraphQL, `******` over REST, `********` in the
  CLI's inherited file), so process code is the only place a value is readable —
  treat code in a child workspace as trusted with its parents' credentials.
  Local runs get the placeholder — put real values in `variables.local.env`.
- **A workspace version never publishes inherited resources** — only owned ones
  get the tag (see Versioning & aliases).

#### Pinning a parent to one of its versions

Each parent link carries an optional **version pin**: a tag, or an alias such as
`stable`, of one of that parent's workspace versions. Without one the parent
resolves **live** — its current code, which is the behaviour above. With one,
that parent contributes **the release, not its working copy**:

- **Only what that version published, with that version's content.** A process
  or module the parent added after cutting the release is simply not there, and
  resolution falls through to the next parent as if the pinned one didn't have
  it. The parent can keep editing — and keep publishing — without moving what
  the child runs.
- **An alias pin follows the alias.** Pin a child to `stable` and re-pointing
  `stable` in the parent moves every child pinned to it, in one operation. This
  is how a base app rolls a fix out to its installations.
- **Dependencies follow the pin too** — the child installs the packages the
  pinned release was cut with, not the ones the parent installs today.
- **Variables always resolve live**, pin or no pin: they have no versioned form.
- **The child cannot pick a version of an inherited resource.** The pin decides
  it. In the web UI an inherited process, module or locale shows its version
  read-only (the pinned tag, or *Live*) and has no Versions tab; publishing and
  aliasing a resource belongs to the workspace that owns it.

The pin is set per parent in the web UI (team settings → **Details** → parent
teams) or in `team.json`; the field reference is in `fcode-cli`. Don't add or
change a pin unless explicitly asked — it decides which release a workspace runs.

On-disk layout, gitignoring, and push/pull behaviour of inherited resources are
in `fcode-cli`.

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
at once — resources inherited through `parentTeams` are never touched. Each
entity's outcome (created / skipped / failed) is recorded in the version's
**manifest**, so re-creating the same tag after a partial failure only publishes
what is still missing.

When a workspace version is published, bare `fcode.import("mod")` /
`fcode.import_module("mod")` calls of workspace-owned modules are pinned to the
tag **inside the published snapshots only** — the working copy is never
modified, and imports that already carry a tag or alias are left untouched.
A workspace version also publishes every owned **locale** and pins
`fcode.i18n` calls the same way — calls already naming a `version` are left
untouched, everything else (bare calls, locale-only options) gets the tag —
in code and in form schemas, so a release ships with its translations frozen —
see `fcode-i18n`.

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
- **A version or alias another workspace pins is protected.** Deleting it is
  rejected, naming the workspaces that would break; they have to unpin first.
  Re-pointing an alias stays allowed — that is how a release is promoted.

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
┃ ┗ 📜 package.inherited.json    # inherited deps (read-only, gitignored); .txt for Py
┣ 📂 i18n
┃ ┣ 📜 <locale>.yaml       #   translations this workspace owns (see fcode-i18n)
┃ ┗ 📜 <locale>.inherited.yaml  # inherited translations (read-only, gitignored)
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
