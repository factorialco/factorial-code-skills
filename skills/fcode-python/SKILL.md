---
name: fcode-python
description: Write Python 3.13 for Factorial Code processes and modules — the main() entry point, fcode.context.parameters, fcode.import_module(), datastore/storage/env helpers, PEP 8 / snake_case, auto-installed dependencies, and return-value formats. Use when creating or editing .py process or module code for Factorial Code (fcode).
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Python

Guidelines for writing Python that runs on Factorial Code. Runtime is
**Python 3.13**. For the platform model (processes, modules, datastore) see
`fcode-core-concepts`.

| Aspect | Guideline |
|---|---|
| Runtime | Python 3.13 |
| Process entry file | `main.py` |
| Entry point | `def main()` |
| Parameters | `fcode.context.parameters` |
| Variables | `os.getenv("X")` or `fcode.env.X` |
| Import a module | `fcode.import_module("module-slug")` |

## Gotchas

- **Define `main()`** as the entry point, but **never call it yourself** —
  Factorial Code invokes it.
- **`fcode.import_module()` names must be hardcoded string literals**, never
  variables: `fcode.import_module("shopify-client")` ✅,
  `fcode.import_module(name)` ❌.
- **Never alias `fcode.i18n`** — call it literally (`fcode.i18n("key")` ✅,
  `t = fcode.i18n` ❌): an aliased call throws "i18n is disabled" at
  runtime. Missing keys resolve to the key itself. See `fcode-i18n`.
- **Datastore stores only strings/numbers** — `json.dumps` objects before
  `set`, `json.loads` after `get`.
- **Use snake_case**, not camelCase; follow PEP 8; add type hints where helpful.
- Wrap the main flow in `try/except`, log the caught error with context via
  `fcode-logs` (see Logging), and raise actionable errors. Don't rely on global
  variables for state — pass it through parameters or return values.
- Never hardcode or log secrets — read them from `os.getenv`.

## Process template

```python
def main():
    parameters = fcode.context.parameters

    # Your code here

    return { "message": "Success!" }
```

## Helpers

```python
import os

# Execution / process / schedule metadata
execution_id = fcode.execution.id
process_id = fcode.execution.process.id
schedule_id = fcode.execution.schedule.id   # when run from a schedule
timezone = fcode.execution.timezone

# Workspace (team) metadata
team_slug = fcode.team.slug

# Environment variables (secrets/config)
api_key = os.getenv("API_KEY")  # or fcode.env.API_KEY

# Import a Factorial Code module (hardcoded name only)
client = fcode.import_module("module-name")
client_v1 = fcode.import_module("module-name", "v1.0.0")  # pinned version tag or alias

# Run another process
fcode.processes.run("process-identifier", options)

# Translations (workspace locales — see fcode-i18n)
greeting = fcode.i18n("greetings.hello", {"name": "Ada"})  # %{name} filled in
locale = fcode.i18n.locale  # the execution's locale
```

## Logging

Log through the shared `fcode-logs` module — level-gated logging inherited by
every workspace. It reads the `LOG_LEVEL` team variable
(`debug | info | warn | error`, default `info`) and forwards to `print` (stdout
for debug/info, stderr for warn/error), so call sites read like a plain `print`:

```python
log = fcode.import_module("fcode-logs")

log.info("sync started", process_slug)  # stdout when LOG_LEVEL ≤ info
log.debug(request_payload)               # stdout only when LOG_LEVEL=debug
log.warn("token missing — skipping")      # stderr when LOG_LEVEL ≤ warn
log.error("sync failed", str(err))         # stderr — always emitted
```

**Be verbose.** Log the start/end of the main flow, every external call, and each
major decision at `info`; dump payloads and intermediate state at `debug` (free
in production — gated off unless `LOG_LEVEL=debug`). **Always log inside `except`:**
record the error with context — what operation, which inputs — at `error` before
re-raising, or at `warn` for a recovered/skipped path. `error` is always emitted.

Set `LOG_LEVEL=debug` in a local or dev workspace to trace a full run; production
stays at `info`. Never log secrets.

## Dependencies

External pip packages install automatically — just `import` them. When the
package name differs from the import name, declare it with `@add-package`:

```python
# @add-package requests
import requests
```

## Datastore & storage

```python
import json, os

# Datastore (strings/numbers only)
fcode.datastore.set("key", "value")
fcode.datastore.set("key", json.dumps({ "name": "John", "age": 30 }))
value = fcode.datastore.get("key")
fcode.datastore.delete("key")

# Storage (files)
local_path = os.path.join(os.environ.get("TMP_DATA_DIR"), "localfile.txt")

with open(local_path, "rb") as f:
    obj = fcode.storage.upload("path/myfile.txt", f)

objects = fcode.storage.list()

content = fcode.storage.download("path/myfile.txt")
with open(local_path, "wb") as f:
    f.write(content)

# Signed download URL — { "url", "expiresAt" }. A real HTTPS link in the
# cloud, a file:// URL locally (same shape, no special-casing).
signed = fcode.storage.create_signed_url("path/myfile.txt")

fcode.storage.delete("path/myfile.txt")
```

**Local disk:** write temp files under `os.environ.get("TMP_DATA_DIR")`.

**Form file uploads:** a form file field (`"ui:widget": "file"`) is uploaded to
Storage before execution and arrives as an `fcode.storage://…` reference (a list
if multiple files). Strip the prefix to download:

```python
parameters = fcode.context.parameters
uploaded_path = parameters.get("inputFile")
content = fcode.storage.download(uploaded_path.replace("fcode.storage://", ""))
```

## Variables & schedules

Read/write team variables and manage process schedules at runtime — scoped to
your own team, no API token needed (like datastore/storage):

```python
# Team variables (config/secrets)
# Default is SENSITIVE: fcode.variables.set(key, value) creates a sensitive
# (masked, immutable-sensitivity) variable. Pass sensitive=False for plain
# config values.
fcode.variables.set("API_KEY", "secret")  # sensitive by default
fcode.variables.set("BASE_URL", "https://api.acme.com", sensitive=False)  # required for non-secret config
v = fcode.variables.get("API_KEY")  # TeamVariable (has resolving_team_slug) or None
all_vars = fcode.variables.list()   # includes variables inherited from parents
fcode.variables.delete("API_KEY")   # no-op on an inherited variable

# Schedules (cron or one-off date_time) for a process
schedule = fcode.schedule.create(
    "my-process",
    cron="0 0 6 * * SUN",  # or: date_time="2026-04-24T12:30:00.000"
    parameters={"foo": "bar"},
    allow_concurrent_executions=False,  # optional
)
schedules = fcode.schedule.list(process_id=fcode.execution.process.id)
fcode.schedule.pause(schedule["id"])
fcode.schedule.resume(schedule["id"])
fcode.schedule.delete(schedule["id"])
# delete every schedule for a process (pass the process UUID)
fcode.schedule.delete_for_process(fcode.execution.process.id)
```

`fcode.variables.set/delete` only persist server-side; they are not reflected in
`fcode.env` within the same run (`fcode.env` is a snapshot taken at start).

### Inherited variables

`list()`/`get()` return the variables the workspace **inherits from its parent
workspaces** alongside its own (model in `fcode-core-concepts`). An inherited one
carries `resolving_team_slug` naming the workspace it comes from; it is readable
and usable exactly like an own variable, but it is owned elsewhere:

- **`set()` on an inherited key creates an override in this workspace** rather
  than updating the parent's variable. That is the only way to change the value
  from here — updating by the parent's id would `404`.
- **`delete()` on an inherited key does nothing** (silently). Only variables this
  workspace owns can be deleted. An uninstall process cleaning up
  `fcode.variables.delete("API_KEY")` therefore leaves a parent's credential
  intact — which is what you want, but check `resolving_team_slug` if the process
  must report what it actually removed.
- `fcode.env.set_env_var` / `del_env_var` behave the same way.

An older platform that doesn't report `resolvingTeamSlug` makes every variable
look owned, so the pre-inheritance behaviour is preserved.

## Sending email

Send email with the built-in `fcode.send_mail` — no SMTP setup required. The mail
server and credentials live in the executor manager, never in your process.

```python
info = fcode.send_mail(
    to="user@example.com",   # string or list[str]
    subject="Report ready",
    text="Plain-text body",  # provide text, html, or both
    html="<b>HTML body</b>",
)
# info => { "messageId", "accepted", "rejected" }
```

- The `From` address is fixed by the platform; a `from` you pass is ignored.
- Each execution can send up to 3 emails by default; once the limit is reached, further calls throw.
- Locally (`fcode run`) there is no manager, so the email is logged, not sent.

## Return values

```python
# Standard
return { "message": "Success!" }

# Custom HTTP status (webhooks)
return { "status": 404, "body": { "message": "Not found" }, "headers": { "Content-Type": "application/json" } }

# Transient (not persisted in execution results)
return { "transient": True, "data": sensitive_data }
```
