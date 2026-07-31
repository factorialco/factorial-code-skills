---
name: fcode-cli
description: Use the Factorial Code CLI (fcode) for local development and cloud sync — the pull → add → dependencies:install → run → push flow, the local webhook/forms server (fcode http), when to run each command, --force safety, worked examples, and the workspace config files (process metadata.json with webhook authMode/auth and form authMode/appRole, team.json with the workspace webhookAuth, variables.meta.json). Use when running fcode CLI commands, testing a Factorial Code process locally, deploying/syncing Factorial Code (fcode) resources to the cloud, or activating and protecting a process's webhook or form settings.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — CLI

The `fcode` CLI develops and tests processes locally and syncs them with
Factorial Code Cloud. For the platform model see `fcode-core-concepts`.

## Command flow

When making and testing changes:

1. **`fcode pull`** *(optional, first)* — if the cloud may have changed, sync
   down so you work with the latest version.
2. Edit local files (processes, modules, variables, dependencies).
3. **`fcode add`** — **only when you created NEW resources** (new process,
   module, dependency, or variable). Skip for edits to existing code.
4. **`fcode dependencies:install`** — if you changed
   `dependencies/package.json` or `dependencies/requirements.txt`.
5. **`fcode run <process-slug>`** — execute the process locally to test.
6. **`fcode push`** — deploy to cloud when ready.

## Gotchas

- **Never run `fcode run` before `fcode add`** when the process (or other
  resource) was *just created* — you'll hit "Local process not found".
- **`fcode add` is only for NEW resources.** For edits to existing process/module
  code or variables, go straight to `fcode push`.
- **`--force` (on `push`/`pull`) overwrites the other side.** Both commands fail
  when local and cloud diverge; only use `--force` with explicit user
  confirmation.

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
```

Prerequisite: run `fcode add` first if the resource was just created.

### `fcode http`

Starts a local HTTP server (`--port`, default `3000`) that replicates the cloud
webhook environment and also serves the workspace's form schemas, so
webhook-triggered processes and forms can be exercised without deploying.

`--auth-user` / `--auth-password` protect the **whole** local server with basic
auth. Per-process webhook auth is separate, and enforced exactly as the cloud
does it: the server reads `webhook.authMode` from the process's `metadata.json`,
resolves `TEAM` against `webhookAuth` in `team.json`, and requires the named
variable's value in the configured header — `Bearer <token>` in `Authorization`,
the raw value in any other header. Values come from the workspace variables
(`variables.env`, overridden by `variables.local.env`). Header lookup is
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

## Team settings — `team.json`

A singleton file at the workspace root holding team-level settings. Synced by
`fcode team:pull` / `team:push` / `team:status`, and included in plain
`fcode push` / `pull` (pushed last, so a referenced error-handler process
exists first).

| Field | Type | Meaning |
|---|---|---|
| `parentTeamSlugs` | string[] | Teams this workspace inherits resources from |
| `zoneId` | string, optional | Team timezone (e.g. for schedules) |
| `errorHandlerConfig` | object, optional | `{ "processSlug": "<slug>", "tag": null }` — process invoked when an execution errors |
| `webhookAuth` | object, optional | `{ headerName?, variableKey }` — the configuration every `authMode: TEAM` webhook inherits |

```json
{
  "parentTeamSlugs": ["base-app"],
  "zoneId": "Europe/Madrid",
  "errorHandlerConfig": { "processSlug": "error-handler", "tag": null },
  "webhookAuth": {
    "headerName": "x-factorial-wh-challenge",
    "variableKey": "FACTORIAL_CHALLENGE_TOKEN"
  }
}
```

The error handler is referenced by **slug** (not id) so `team.json` is
portable across teams; the CLI resolves it to the cloud id on push.

`webhookAuth` is one shared configuration for the whole workspace, so a token used
by several webhooks is named — and rotated — in one place. Two things about it:

- **It is per-workspace and not inherited through `parentTeamSlugs`.** Auth is
  resolved against the workspace addressed in the webhook URL, not the one that
  owns the code, so an app inheriting a webhook process from a base app still
  needs its own `webhookAuth` entry.
- **Removing the object and pushing clears the cloud configuration**, which makes
  every webhook inheriting it reject all calls.

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
  `********` into `variables.env`. Don't replace the placeholder there — put
  the real value in `variables.local.env` for local runs.
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
  puts it in `variables.local.env` (or shares it, per their preference).
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
