---
name: fcode-ui-triggers
description: Surface a Factorial Code process as a button inside the Factorial product UI — the `factorial.uiTrigger` settings in a process's metadata.json (location id, label, icon; `awaitResult` at the block's top level) and the legacy `uiTrigger` block they replace, what the process receives when a user clicks, what a synchronous trigger returns, form-backed triggers and their file uploads, i18n labels, the icon allowlist, and how Factorial's `FactorialCodeTrigger` component renders them. Use when an app process should appear as an action button on a Factorial page, or when wiring the button entry point of a Factorial Action ("Trigger from Factorial" in the console).
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — UI triggers

A UI trigger exposes a marketplace app's process as a **button inside
Factorial** (a page header, an actions dropdown, …). Factorial pages declare
*locations*; an installed app's process claims a location, and Factorial renders
one button per claiming process. Clicking it runs the process — or opens its
form. It is one entry point of a **Factorial Action**: its settings live in the
`factorial.uiTrigger` sub-block of the process's `metadata.json`, with
`awaitResult` at the top level of the `factorial` block (the whole block, the
legacy mirror and the Factorial One / backend entry points are in
`fcode-factorial-actions`; field reference in `fcode-cli`).

## Gotchas

- **Only installed marketplace apps get buttons.** Factorial lists the triggers
  of the apps the company has installed, read from their deploy workspaces. A
  process in a plain workspace never shows up, and a trigger reaches customers
  the way the rest of the app does: dev → prod → deploy through a release.
- **The location id must come from the Factorial team that owns the page.**
  Locations are declared in Factorial's code (e.g. `calendar.header.admin`), not
  on the platform. The platform accepts any non-blank `locationId` (≤ 200
  chars) — an unknown one is silently dropped from the listing, so a typo means
  "no button", not an error.
- **The location decides the contract**, not the process: which context params
  are forwarded to the process, which keys of a synchronous result reach the
  page, and whether the location is `single` (one installed app at a time).
  Ask for those three things along with the id.
- **`awaitResult` lives on the `factorial` block and defaults to `true`
  (synchronous).** The legacy `uiTrigger.awaitResult` defaulted to
  fire-and-forget, and the mirror copies the new value over it — so a button
  moved into `factorial` runs synchronously unless you write
  `"awaitResult": false`. Keep it `true` only for fast work (< 50 s) whose
  outcome the user must see; with `false` the click queues an execution, the
  user sees "action started" and the return value is never shown.
- **A form-enabled process opens its form instead of running.** With
  `factorial.form.enabled` (or the legacy `form.enabled`) on, the button opens
  the form in a dialog inside Factorial and `awaitResult` is ignored.
- **A trigger-opened form goes through Factorial, not straight to the
  platform.** The dialog reads the schema and submits it through Factorial's own
  backend, which carries the clicking user's identity. The form's own
  `authMode` still governs anyone reaching it by its public URL (`fcode-forms`);
  it does not govern the dialog.
- **`company_id`, `triggered_from_location` and `access_id` are trustworthy on
  both paths.** Factorial injects them server-side on every invocation, form or
  not, and the location's forwarded params overwrite whatever the browser sent.
  Authorizing on them is safe. (This was not true of the old form path, which
  passed them as client-editable pre-filled fields.)
- **Who may click is decided by `requiredPolicies`, and it is enforced.**
  Factorial checks `factorial.requiredPolicies`
  (`fcode-factorial-actions`) against the clicking user before it runs anything,
  on top of a policy-scoped read of the location's resource. An action with no
  policies listed is open to anyone who can reach the button.
- **Uncaught errors show a generic message.** A throw / crash reaches the user as
  "The action could not be completed" with no detail. Return `{ errors: [...] }`
  for anything the user should read.
- **Changes take up to five minutes to appear** — Factorial caches the trigger
  listing in the browser.

## Declare a trigger

In `processes/<slug>/metadata.json`, then `fcode push` (or the **Trigger from
Factorial** section of the process page in the console):

```json
{
  "name": "Sync report",
  "tags": ["acme"],
  "factorial": {
    "enabled": true,
    "uiTrigger": {
      "enabled": true,
      "locationId": "compensations.cycle.header",
      "label": "Sync to Acme",
      "icon": "Refresh"
    }
  }
}
```

| Field | Meaning |
|---|---|
| `factorial.enabled` | Master switch of the whole action; the button needs it on |
| `factorial.awaitResult` | Defaults to `true`: run synchronously and show the outcome. Write `false` for fire-and-forget |
| `uiTrigger.enabled` | Turns the button on. `locationId` is required when `true` |
| `uiTrigger.locationId` | The Factorial location the button renders at, as given by the page's owning team |
| `uiTrigger.label` | Button text. Plain text, or `fcode.i18n("key")` tokens (below) — the only i18n field of the block |
| `uiTrigger.icon` | One of the allowlisted names (below); omit for a text-only button |

The legacy top-level `uiTrigger` block (`enabled`, `locationId`, `label`,
`icon`, `awaitResult`) still round-trips and is kept mirrored with
`factorial.uiTrigger` — an old file keeps working, and `fcode pull` still
writes `"uiTrigger": { "enabled": false }` on every process. Write `factorial`
in new work; when both are present `fcode push` sends `factorial` (mirror rules
in `fcode-factorial-actions`).

An app may declare several triggers at one location — each renders its own
button — but a location marked `single` in Factorial admits **one installed
app**: installing a second app that claims it fails with a conflict naming the
first. Don't claim a `single` location from an app meant to coexist with others.

## What the process receives

The click runs the process with `fcode.context.parameters` set to:

- the location's **forwarded context params** — e.g. the id of the record the
  page shows (`cycle_id`), exactly the keys the location allowlists;
- `company_id` — the Factorial company whose user clicked (trusted: injected
  under a shared secret, overwriting anything the browser sent);
- `triggered_from_location` — the location id (trusted, same way);
- `access_id` — the Factorial access (the clicking user's membership in that
  company) as a string (trusted, same way).

A context-less location (a header button with no record in scope) forwards no
params at all — only the three trusted keys. The clicking user's locale is passed
as the execution locale, so `fcode.i18n` in the process speaks their language
(`fcode-i18n`).

```javascript
const { company_id, access_id, triggered_from_location, cycle_id } = fcode.context.parameters;
if (!cycle_id) throw new Error("cycle_id is required");
```

## Return a result (synchronous triggers)

With `awaitResult: true` Factorial waits for the execution and hands the page
**whatever you returned, unchanged**. There is no envelope to wrap it in, and
nothing is filtered out on the way.

```javascript
// Success — the whole object reaches the page's onSuccess handler.
return { synced: 42, report_url: "https://acme.example/reports/7" };
```

- **A success status is always a success**, whatever the body says. Returning an
  object with an `errors` key in it does not make it a failure.
- **Signal an expected failure with an error status**, not with the body. Throw
  `fcode.error(...)`, or return an error status from an HTTP-shaped process; the
  body travels intact so the page can read your message.
- A throw with no message, or exceeding the synchronous budget (about 50 s; the
  execution may still finish in the background) → a generic error the user can't
  act on.

Matching what the entry point expects is the process author's job: a form reads
its own conventions (`message`, `formErrors`, `nextProcessId`) while a plain
button just shows a toast, so a process exposed as both does not get one
portable result shape. `result_keys` on the location is informational — it
describes what the page intends to read, and nothing filters by it.

With `awaitResult` off the return value is only visible in the execution log;
long work belongs there, or behind `fcode.processes.run(...)` from a quick
synchronous trigger (see `fcode-javascript` / `fcode-python`).

## Form-backed triggers

Enable the form as usual (`fcode-forms`) and the button opens it in a dialog
inside Factorial, rendered with the f0 theme, pre-filled with the forwarded
params. Whether a click opens the dialog or runs the process comes from the
action's entry points: declaring both `form` and `uiTrigger` opens the form,
`uiTrigger` alone runs it.

The submission goes through Factorial's backend, which injects `company_id`,
`triggered_from_location` and `access_id` server-side — declare them in
`parametersSchema.json` (a `hidden` widget) if the process reads them. The
result follows the form conventions (`message`, `formErrors`, `nextProcessId`,
…); a multistep form follows each step's `nextProcessId` inside the same dialog.

```json
{
  "name": "Export cycle",
  "factorial": {
    "enabled": true,
    "form": { "enabled": true },
    "uiTrigger": { "enabled": true, "locationId": "compensations.cycle.header", "label": "Export…", "icon": "Download" }
  }
}
```

### Files in a trigger form

A file field works, and the file does not travel through Factorial. Declare the
parameter with `x-fcode-file` listing the extensions it takes:

```json
"msj_file": {
  "title": "FIE file",
  "type": "string",
  "x-fcode-file": { "accept": [".msj"] },
  "ui": { "ui:widget": "file", "ui:options": { "accept": ".msj" } }
}
```

On submit, Factorial asks the platform for an upload grant per file, PUTs the
file straight to Factorial Code storage, and sends the resulting
`fcode.storage://` path as the parameter value. The process reads it as it would
any stored file (`fcode-javascript` / `fcode-python`).

- The grant is issued **at submit, not with the schema** — the platform checks
  the real file name against `accept` and signs its content type, and neither is
  known before the user picks a file. A name outside `accept` is refused there.
- Uploaded files are **removed once they are 48 hours old**. Copy anything the
  process needs for longer elsewhere in storage.

## Labels and i18n

`label` may embed `fcode.i18n("key")` tokens — the same tokens a form schema
uses — resolved with each user's locale from the workspace's locale files, with
the primary-locale fallback and never a blank label (model, files and syntax in
`fcode-i18n`):

```json
"factorial": {
  "enabled": true,
  "uiTrigger": { "enabled": true, "locationId": "calendar.header.admin", "label": "fcode.i18n(\"acme.sync.button\")" }
}
```

## Icons

Factorial renders only these names (case-insensitive); anything else falls
back to a text-only button, and the console shows the same list as a picker:

`Bell`, `Calendar`, `Chart`, `Document`, `Download`, `ExternalLink`, `Globe`,
`Graph`, `Link`, `Play`, `Refresh`, `Send`, `Settings`, `Sparkles`, `Star`,
`Upload`.

## How Factorial renders them (for Factorial engineers)

The Factorial side is generic — no per-app code. In the `factorial` monolith:

- A location is a YAML file under a component's `app/fcode_locations/`
  (`id`, optional `resource` + `param_key` for the record in scope,
  `forward_params`, `result_keys`, `single`, `legacy_ui`), validated by a CI
  spec. `calendar/header_admin.yml` is the reference.
- The page renders
  `<FactorialCodeTrigger locationId="…" params={{ cycle_id }} onSuccess onError />`
  from `frontend/src/modules/factorialCode/components/FactorialCodeTrigger`. It
  draws one button per claiming trigger (F0 `outline` button, or the legacy
  design-system button for `legacy_ui` locations), renders nothing while loading
  or when no installed app claims the location, and owns the form dialog. The
  underlying hooks (`useFactorialCodeTriggers` for the listing,
  `useFactorialCodeTriggerActivation` for the click) are exported for
  data-driven hosts such as dropdown menus.
- Clicks go through `FactorialCode::Interactors::InvokeAction`: it checks the
  action's `requiredPolicies` (`RequiredPoliciesAuthorization`, an OR of AND
  groups) and a policy-scoped read of the location's `resource`, forwards only
  `forward_params`, injects `company_id` / `access_id` /
  `triggered_from_location` over anything the browser sent, and returns the
  process's answer unchanged. The SPA addresses an action by its opaque
  `action_id`; no process slug reaches the browser. Everything is gated by the
  Factorial Code feature flag.

App developers don't touch any of this; they need the location's id and
contract from the team that owns the page.

## Checklist before shipping

1. Location id, forwarded params and `single`-ness confirmed with the owning
   Factorial team.
2. `factorial.awaitResult` decided explicitly — `true` (the default) only for
   fast, user-visible outcomes; `false` otherwise.
3. `factorial.requiredPolicies` filled in, or deliberately left empty because
   the button is open to anyone who can reach it.
4. A form-backed trigger declares the pre-filled keys it reads, and any file
   parameter carries `x-fcode-file`.
5. Tested from a dev installation in Factorial, then promoted to `prod-` and
   released so the deploy workspaces pick it up (`fcode-release`, `fcode-ama`).
