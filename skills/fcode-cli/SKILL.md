---
name: fcode-cli
description: Use the Factorial Code CLI (fcode) for local development and cloud sync — the pull → add → run → push flow, the local webhook/forms server (fcode http), workspace versions and aliases (declared in team.json and synced by push/pull, the stable alias, version_tag), and the workspace config files (process metadata.json, team.json, the three variables files, variables.meta.json). Use when running fcode CLI commands, testing a process locally, syncing to the cloud, managing versions or aliases, overriding an inherited team variable, or configuring a process's webhook or form settings.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — CLI

The `fcode` CLI develops and tests processes locally and syncs them with
Factorial Code Cloud. For the platform model see `fcode-core-concepts`.

## Command flow

First time on a machine: install with `pnpm install -g @factorialco/fcode-cli`,
then **`fcode clone <workspace-slug>`** to bring a cloud workspace down into a
`<workspace-slug>/` folder (it also installs these agent skills; skip that with
`--skipSkillsSetup`). The "How to build locally" guide on an App's Development
tab shows the exact clone command for its dev workspace — copy it from there,
the slug is an encoded token, not the App's UUID.

When making and testing changes in an already-cloned workspace:

1. **`fcode pull`** *(optional, first)* — if the cloud may have changed, sync
   down so you work with the latest version.
2. Edit local files (processes, modules, variables, dependencies).
3. **`fcode add`** — **only when you created NEW resources** (new process,
   module, dependency, or variable). Skip for edits to existing code.
4. **`fcode dependencies:install`** — if you changed
   `dependencies/package.json` or `dependencies/requirements.txt`.
5. **`fcode run <process-slug>`** — execute the process locally to test.
6. **`fcode push`** — deploy to cloud when ready.

Pushing updates the **current** (unversioned) code only: consumers pinned to the
`stable` alias — which webhook URLs and form embeds should always be — keep
running the released version until the alias moves. Releases (publishing a
workspace version and re-pointing `stable`) happen separately, normally from the
web UI (see below).

## Gotchas

- **Never run `fcode run` before `fcode add`** when the process (or other
  resource) was *just created* — you'll hit "Local process not found".
- **`fcode add` is only for NEW resources.** For edits to existing process/module
  code or variables, go straight to `fcode push`.
- **`--force` (on `push`/`pull`) overwrites the other side.** Both commands fail
  when local and cloud diverge — `team:pull` included, which refuses to wipe an
  unpushed `team.json`. Only use `--force` with explicit user confirmation.
- **Editing `versions` / `aliases` in `team.json` releases on push**: adding a tag
  publishes that workspace version, removing one deletes it from the cloud
  (cascading), re-pointing an alias is a release or rollback. Same rule as
  `team:versions:*` / `team:aliases:*` — **don't touch unless explicitly asked.**
- **Never edit `variables.inherited.env`.** It holds the parent workspaces'
  variables, is regenerated on pull, and `fcode push` skips it with a warning.
  Overriding an inherited value means adding the key to `variables.env`.

## Commands

### `fcode add`

Registers **new** local resources with the CLI so they can be run or deployed.
Run after creating a new process/module/dependency/variable, before `run`/`push`.
Not needed after only editing existing resources.

### `fcode dependencies:install`

Installs dependencies into the local workspace. Run after changing
`dependencies/package.json` or `dependencies/requirements.txt`.

### `fcode run <process-slug> --parameters <filepath | json>`

Executes a process locally for development/testing. Uses `variables.env` /
`variables.local.env` and the given parameters (or the process's
`parameters.json` by default). Shows logs, results, and errors.

```sh
fcode run my-process --parameters '{"key": "value"}'
fcode run my-process --parameters ./params.json
fcode run my-process                                  # default parameters.json
fcode run my-process --locale pt-BR                   # resolve fcode.i18n in that locale
```

Prerequisite: run `fcode add` first if the resource was just created.

`--locale` resolves `fcode.i18n` against the local `i18n/` files exactly as the
cloud does — see `fcode-i18n` (including why a typo in the flag silently
resolves every key to itself).

### `fcode http`

Starts a local HTTP server (`--port`, default `3000`) that replicates the cloud
webhook environment and also serves the workspace's form schemas, so
webhook-triggered processes and forms can be exercised without deploying.

`--auth-user` / `--auth-password` protect the **whole** local server with basic
auth. Per-process webhook auth is separate, and enforced exactly as the cloud
does it: the server reads `webhook.authMode` from the process's `metadata.json`,
resolves `TEAM` against `webhookAuth` in `team.json`, and requires the named
variable's value in the configured header — `Bearer <token>` in `Authorization`,
the raw value in any other header. Values come from the workspace variables in
the precedence every local run uses — `variables.inherited.env`, overridden by
`variables.env`, overridden by `variables.local.env` (below). Header lookup is
case-insensitive, but a repeated header is rejected rather than joined. A missing
header, a malformed bearer value, an undefined variable, a mismatch, or a
`TEAM` webhook whose workspace configuration is absent all get a `403`, with the
reason printed to the console.

Two things to watch locally:

- A **secret** variable pulls down as the `********` placeholder, so local auth
  only accepts `********` until the real value is in `variables.local.env`.
- `--auth-user` / `--auth-password` consume the `Authorization` header, so they
  can't be combined with a webhook expecting its credential there. The server
  warns about this at startup.

### `fcode pull`

Downloads the latest processes, modules, variables, and dependencies from the
cloud, overwriting local files to match. Run before starting work if others may
have changed cloud resources, or to discard local changes. `--force` only with
user confirmation.

### `fcode push`

Uploads local changes to the cloud. Run after local changes (run `fcode add`
first only if you created new resources); recommended to `fcode run` first.
`--force` only with user confirmation.

### `fcode team:pull` / `team:push` / `team:status`

Sync the workspace-level settings in `team.json` on their own: `team:pull` writes
the cloud settings into the file, `team:push` applies the file, and `team:status`
reports whether they changed locally, in the cloud, or both. Plain `fcode pull` /
`fcode push` include them too, running them **last** so a referenced error-handler
process slug resolves against processes that already exist.

The `versions` / `aliases` fields sync like the rest of the file: `team:push`
diffs them against the cloud and applies the difference — publishing versions
listed locally but not yet in the cloud (carrying their `comment`), asking per
version before deleting one removed from the file (the deletion **cascades**;
`--force` skips the prompt), and creating, re-pointing or deleting aliases to
match. `team:status` compares them too, and `team:pull` refuses to overwrite a
`team.json` with unpushed local edits.

### `fcode team:versions:*` / `fcode team:aliases:*`

Workspace versioning publishes a version of the **whole workspace**: every
process and module the team owns gets a version with the same tag, and bare
module imports are pinned to it inside the published snapshots (model in
`fcode-core-concepts`). Releases normally happen from the web UI (team
settings → **Versions** tab) — these commands are the imperative equivalent, and
editing `team.json`'s `versions` / `aliases` then pushing is the declarative one.
**Don't create versions or move `stable` unless explicitly asked.**

```sh
fcode team:versions:create v1.0.0 --comment "First stable release"
fcode team:versions:list
fcode team:versions:delete v1.0.0     # cascades; asks confirmation unless --force

fcode team:aliases:set stable v1.0.0  # create or re-point; rollback = older tag
fcode team:aliases:list
fcode team:aliases:delete stable
```

- **`team:versions:create`** skips entities already carrying the exact tag and
  reports a per-entity summary (created / skipped / failed — the version's
  manifest), then pulls so the `versions/<tag>/` folders and `team.json` refresh
  locally. Re-running the same tag after a partial failure only publishes what
  is still missing.
- **`team:versions:delete` cascades**: every owned process/module version with
  the tag is deleted, together with the aliases, executions, and schedules
  referencing them.
- **`team:aliases:set` upserts** — it creates the alias or re-points an existing
  one on every owned entity that has the target tag published (entities without
  it are skipped and reported). Re-pointing `stable` at an older tag **is** the
  rollback: every consumer pinned to `stable` switches in one operation.

### `fcode i18n:*`

`i18n:pull` / `i18n:push` / `i18n:status` / `i18n:add <locale>` /
`i18n:remove <locale>` / `i18n:reset` sync the workspace's translation files —
`i18n/<locale>.yaml`, plus the read-only, gitignored
`i18n/<locale>.inherited.yaml` that `pull` writes. Aggregate `fcode pull` /
`push` / `status` include locales already. File format, inheritance model, and
the internationalization workflow in `fcode-i18n`.

## Process metadata — `metadata.json`

Each process folder holds `processes/<slug>/metadata.json` — the source of
truth for the process's name, description, tags, triggers, and settings. It
round-trips with `push`/`pull`: edit the file and `fcode push` to change these
settings in the cloud, no dashboard needed. Changes show as 🔺 modified in
`fcode status`.

| Field | Type | Meaning |
|---|---|---|
| `name` | string | Display name (defaults to the slug) |
| `description` | string, optional | Process description |
| `tags` | string[] | Tags (defaults to `[]`) |
| `webhook` | object, optional | Webhook trigger: `enabled` (boolean) turns the process's webhook endpoint on; `authMode` (`NONE` \| `TEAM` \| `CUSTOM`) says how callers authenticate — public, inheriting the workspace `webhookAuth` from `team.json`, or its own; `auth` (`{ headerName?, variableKey }`, only with `CUSTOM`) names the header and the team variable holding the expected token |
| `form` | object, optional | Form settings: `enabled` (boolean) is the Forms flag (see `fcode-forms`); `authMode` (`FACTORIAL` \| `NONE`) restricts who may open the form; `appRole` marks the process's role in a marketplace app: `INSTALL`, `SETTINGS`, `USER_FACING_FORM`, or `UNINSTALL` |

```json
{
  "name": "Order sync",
  "description": "Syncs Shopify orders into Factorial",
  "tags": ["integration", "shopify"],
  "webhook": {
    "enabled": true,
    "authMode": "CUSTOM",
    "auth": { "variableKey": "SHOPIFY_WEBHOOK_TOKEN" }
  },
  "form": { "enabled": false }
}
```

A webhook that inherits the workspace configuration carries
`"webhook": { "enabled": true, "authMode": "TEAM" }`, and a public one only
`"webhook": { "enabled": true }`.

```json
{
  "name": "Connect your account",
  "tags": ["setup"],
  "form": { "enabled": true, "authMode": "FACTORIAL", "appRole": "INSTALL" }
}
```

Notes:

- **`webhook.auth.variableKey` stores only the variable *name*, never a token** —
  so the file is safe to commit. The variable doesn't have to exist yet; until it
  does, every call to the webhook is rejected with `403`. Both plain and secret
  variables work.
- **`authMode: TEAM` inherits `webhookAuth` from `team.json`.** When that
  configuration is missing, the webhook rejects every call — it never reads as
  public. Through MCP this matters: an agent can set `authMode: TEAM` but there is
  no team-settings tool, so the configuration has to exist already (set it in
  `team.json` and `fcode team:push`).
- **`webhook.auth.headerName` defaults to `Authorization`**, whose value must be
  `Bearer <token>`; any other header carries the raw variable value. Valid names
  are RFC 7230 token characters, at most 64 of them, and `Cookie`, `Host` and the
  `Fcode-` prefix are rejected. Prefer the default: a bespoke header loses the
  redaction proxies and log pipelines give `Authorization`. Use one only when the
  sender can't set `Authorization` — Factorial's own webhook sender, which puts
  its token in `x-factorial-wh-challenge`, is the case in point.
- **`form.authMode`, `form.appRole` and `webhook.authMode` are omitted when they
  are `NONE`**, as is `webhook.auth.headerName` when it is `Authorization`, so a
  plain public form carries only `"form": { "enabled": true }`. To lift protection
  from a protected form or webhook, write `"authMode": "NONE"` **explicitly** —
  omitting the field leaves the stored mode untouched, and sending `auth` without
  `authMode: CUSTOM` is rejected.
- Omit `form.appRole` unless the process belongs to a marketplace app.
- If `metadata.json` is missing, `fcode add` scaffolds
  `{ "name": "<slug>", "tags": [] }`; invalid JSON falls back to those
  defaults with a warning.

### Calling a webhook — pin the version in the URL

The webhook endpoint is
`https://code.factorialhr.com/platform/api/<team-slug>/webhooks/<process-slug>`.
Always pin the version with the `version_tag` query parameter, pointing at the
`stable` alias — subscription systems rarely let you set request headers:

```sh
curl -X POST "https://code.factorialhr.com/platform/api/<team-slug>/webhooks/<process-slug>?version_tag=stable"
```

- `version_tag` takes a version tag (`v1.0.0`) or an alias. Use `stable`: it
  always exists, and releases/rollbacks then happen by moving the alias — the
  external system is never touched. It is equivalent to the `Fcode-Version-Tag`
  header and takes precedence over it. `version_tag`, `async` and `locale` are
  reserved names, stripped before the parameters reach the process (`locale`
  selects the execution's language — see `fcode-i18n`).
- **An unknown or malformed version does not fail the call.** The process runs
  its current version and the platform only logs a server-side warning — a typo
  runs the current version silently. When a run behaves unexpectedly, check the
  execution's version.

## Team settings — `team.json`

A singleton file at the workspace root holding team-level settings. Synced by
`fcode team:pull` / `team:push` / `team:status`, and included in plain
`fcode push` / `pull` (pushed last, so a referenced error-handler process
exists first).

| Field | Type | Meaning |
|---|---|---|
| `parentTeamSlugs` | string[] | Teams this workspace inherits processes, modules **and variables** from (direct parents only, max 5) |
| `zoneId` | string, optional | Team timezone (e.g. for schedules) |
| `errorHandlerConfig` | object, optional | `{ "processSlug": "<slug>", "tag": null }` — process invoked when an execution errors; `tag` pins it to a version tag or alias (`null` = current version) |
| `webhookAuth` | object, optional | `{ headerName?, variableKey }` — the configuration every `authMode: TEAM` webhook inherits |
| `primaryLocale` | string, optional | The workspace's main language: the locale used when a caller names none, and the key-level fallback for untranslated keys (see `fcode-i18n`) |
| `versions` | array, optional | Workspace versions: `{ tag, comment?, createdAt? }` — declaring a new `tag` and pushing publishes it; omit `createdAt`, the cloud fills it in |
| `aliases` | array, optional | Workspace aliases: `{ name, tag }` |

```json
{
  "parentTeamSlugs": ["base-app"],
  "zoneId": "Europe/Madrid",
  "errorHandlerConfig": { "processSlug": "error-handler", "tag": null },
  "webhookAuth": {
    "headerName": "x-factorial-wh-challenge",
    "variableKey": "FACTORIAL_CHALLENGE_TOKEN"
  },
  "versions": [
    { "tag": "v1.0.0", "comment": "First stable release", "createdAt": "2026-08-01T10:00:00" }
  ],
  "aliases": [{ "name": "stable", "tag": "v1.0.0" }]
}
```

The error handler is referenced by **slug** (not id) so `team.json` is
portable across teams; the CLI resolves it to the cloud id on push.

`versions` and `aliases` are **synced state**, not a read-only mirror: they count
towards the content hash (edits show as modified in `fcode status`) and
`team:push` applies them, with the diff semantics above. Only the set of tags and
the `name` → `tag` pairings are compared — a published version's `comment` can
no longer be changed, and `createdAt` is server-assigned.

`webhookAuth` is one shared configuration for the whole workspace, so a token used
by several webhooks is named — and rotated — in one place. Two things about it:

- **It is per-workspace and not inherited through `parentTeamSlugs`.** Auth is
  resolved against the workspace addressed in the webhook URL, not the one that
  owns the code, so an app inheriting a webhook process from a base app still
  needs its own `webhookAuth` entry.
- **Removing the object and pushing clears the cloud configuration**, which makes
  every webhook inheriting it reject all calls.

## The three variables files

Team variables live in three `.env` files at the workspace root:

| File | Holds | Synced |
|---|---|---|
| `variables.env` | The variables this workspace **owns** | Committed; pushed and pulled |
| `variables.inherited.env` | The variables inherited from **parent workspaces** (`parentTeamSlugs`) | Pull-only; **gitignored** (the CLI adds the entry) |
| `variables.local.env` | Local-only overrides | Never pushed, never pulled |

**Resolution order for a local run** (highest wins), matching what the cloud
does: `variables.local.env` → `variables.env` → `variables.inherited.env`.

Precedence is decided by which file **declares** a key, not by its value — so
blanking a key in `variables.env` overrides the inherited variable with an empty
string rather than falling through to the parent.

### Overriding an inherited variable

**Adding the key to `variables.env` *is* the override.** From that point the CLI
treats it as this workspace's own variable: `fcode status` shows it as new,
`fcode push` creates it here, and `fcode variables:add` offers it.

- **Don't edit `variables.inherited.env`** — it is regenerated on every pull, and
  `fcode push` skips inherited variables with a warning. Editing one only warns.
- **Don't copy a parent's variables into a child workspace** to "make them
  available" — they already resolve. Only add a key when this workspace genuinely
  needs a different value. (Workspaces provisioned before inheritance existed may
  still hold such copies, which now shadow the parent — including untouched
  `********` placeholders shadowing a secret that would otherwise resolve. Flag
  those to the user rather than deleting them.)
- Deleting your override (removing the key from `variables.env` and pushing)
  brings the parent's value back.

`fcode variables:status` grows an **inherited** column showing the source
workspace (`🔗 <slug>`) when any variable is inherited; the column is hidden
otherwise. Model and web-UI behaviour in `fcode-core-concepts`; the runtime
`fcode.variables` behaviour in `fcode-javascript` / `fcode-python`.

The file names are settings (`variablesFileName`,
`inheritedVariablesFileName`, `localVariablesFileName`) — assume the defaults
above unless the workspace says otherwise.

## Variable sensitivity — `variables.meta.json`

A workspace-root file mapping each variable to its sensitivity flag:

```json
{
  "ACME_API_KEY": { "isSensitive": true },
  "ACME_BASE_URL": { "isSensitive": false }
}
```

- Create a sensitive variable with `fcode variables:add --sensitive` (then set
  its value and push); the flag lands here.
- Variables created at runtime with `fcode.variables.set` are **sensitive by
  default** — pass `sensitive: false` (JS) / `sensitive=False` (Python) for
  plain config. See `fcode-javascript` / `fcode-python`.
- **Sensitive values never leave the cloud**: `pull` writes the placeholder
  `********` into `variables.env` — and into `variables.inherited.env` for an
  inherited secret. Don't replace the placeholder in either file — put the real
  value in `variables.local.env` for local runs. Remotely, an inherited secret's
  real value *is* available to executions (see `fcode-core-concepts`); only the
  local copy is masked.
- **`isSensitive` is immutable** once pushed. Editing it in
  `variables.meta.json` is rejected on push (🚫 in `fcode status`) — revert to
  match remote.

### Getting secret values for local runs

When a local run (`fcode run`, a discovery script) needs a real secret value
that isn't in `variables.local.env` yet, ask the user to provide it. If they
prefer not to share the value with the agent, ask them to add the
`KEY=value` line to `variables.local.env` themselves — local runs pick it up
without the value ever appearing in the conversation.

- **`variables.local.env` values are never pushed.** Remind the user to also
  create those secret variables manually in the remote demo environment —
  `fcode push` won't carry the values.
- **`FACTORIAL_TOKEN`**: needed **locally only** — the remote environment
  populates it automatically, so don't create it there. To obtain it, the user
  completes the OAuth flow in the Factorial Code app details page, then copies
  the generated token with the copy dropdown option in the OAuth Dev app, and
  puts it in `variables.local.env` (or shares it, per their preference). To run
  in a specific installed company's context instead, copy that `deploy-`
  workspace's token with **Copy FACTORIAL_TOKEN** in the installation's "…"
  menu (Installations page or Dev Marketplace app page) — same file, same
  handling.
- Once obtained, never echo secret values back in output or logs.

## Examples

**Development cycle (new process):**

```sh
fcode add
fcode dependencies:install            # if dependencies changed
fcode run shopify-order-sync --parameters '{"dateFrom":"2024-01-01","dateTo":"2024-01-31"}'
fcode push                            # no need to re-run `add` if nothing new was created
```

**Deploy an existing, tested process:**

```sh
fcode push                            # `add` not needed — process already registered
```
