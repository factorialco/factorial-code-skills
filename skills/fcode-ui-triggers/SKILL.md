---
name: fcode-ui-triggers
description: Surface a Factorial Code process as a button inside the Factorial product UI — the `factorial.uiTrigger` settings in a process's metadata.json (location id, label, icon; `awaitResult` at the block's top level) and the legacy `uiTrigger` block they replace, what the process receives when a user clicks, the `{ data }` / `{ errors }` result envelope for synchronous triggers, form-backed triggers, i18n labels, the icon allowlist, and how Factorial's `FactorialCodeTrigger` component renders them. Use when an app process should appear as an action button on a Factorial page, or when wiring the button entry point of a Factorial Action ("Trigger from Factorial" in the console).
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
- **Trigger-opened forms are not authenticated.** The dialog sends no Factorial
  user token, and forms carry no access restriction of their own — a form behind
  a trigger is public like any other (see `fcode-forms`).
- **Never authorize on `company_id` / `triggered_from_location` / `access_id`
  in a form.** On the *execute* path they are injected server-side and
  trustworthy; on the *form* path they arrive as pre-filled, client-editable
  fields.
- **Who may click is decided by Factorial, not by the block — for now.** Today
  Factorial authorizes a click with a policy-scoped read of the location's
  resource. `factorial.requiredPolicies` (`fcode-factorial-actions`) is stored
  already and replaces the location's policies once the trusted invocation
  path lands; fill it in now.
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

With `awaitResult: true` Factorial waits for the execution and reads the return
value as an envelope:

```javascript
// Success — `data` is shown to the page. Only the keys the location allowlists
// (`result_keys`) get through; anything else is dropped before reaching the browser.
return { data: { synced: 42, report_url: "https://acme.example/reports/7" } };

// Controlled error — the messages render to the user, as plain text.
return {
  errors: [{ code: "missing_mapping", message: "Map the 'Bonus' concept in Acme first." }],
};
```

- `{ data }` → a success toast; the (filtered) `data` reaches the page's
  `onSuccess` handler. Return `{ data: {} }` when there is nothing to hand back.
- `{ errors: [{ code, message }] }` → the user reads your messages. Both fields
  must be strings; entries missing either are ignored. Use it for every
  expected failure (bad configuration, vendor rejection, nothing to do).
- Anything else (no envelope, a bare `{ message }`) still counts as **success**
  with an empty result — the execution finished.
- A throw, a platform `4xx`, or exceeding the synchronous budget (about 50 s;
  the execution may still finish in the background) → a generic error the user
  can't act on.

With `awaitResult` off the return value is only visible in the execution log;
long work belongs there, or behind `fcode.processes.run(...)` from a quick
synchronous trigger (see `fcode-javascript` / `fcode-python`).

## Form-backed triggers

Enable the form as usual (`fcode-forms`) and the button opens it in a dialog
inside Factorial, rendered with the f0 theme, pre-filled with the forwarded
params plus `company_id`, `triggered_from_location` and `access_id` as default
values.
Declare those keys in `parametersSchema.json` (a `hidden` widget) if the process
needs them — and remember they are client-editable there. The submission result
follows the form conventions (`message`, `formErrors`, `nextProcessId`, …); a
`{ data }` envelope in the final step is handed to the page like a synchronous
trigger's.

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

The form is public and the dialog carries no user identity, so authorize inside
the process, never on the forwarded params — see the gotchas above.

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
- Activation goes through `FactorialCode::Interactors::ActivateUiTrigger`
  (authorizes with a policy-scoped read of the location's `resource`, forwards
  only `forward_params`, filters `data` to `result_keys`) and is proxied to the
  Factorial Code dashboard, which resolves the trigger live and runs the
  process. Everything is gated by the Factorial Code feature flag.

App developers don't touch any of this; they need the location's id and
contract from the team that owns the page.

## Checklist before shipping

1. Location id, forwarded params, `result_keys` and `single`-ness confirmed with
   the owning Factorial team.
2. `factorial.awaitResult` decided explicitly — `true` (the default) only for
   fast, user-visible outcomes, and the process returns `{ data }` /
   `{ errors }`; `false` otherwise.
3. A form-backed trigger's form is public and declares the pre-filled keys it
   reads.
4. Tested from a dev installation in Factorial, then promoted to `prod-` and
   released so the deploy workspaces pick it up (`fcode-release`, `fcode-ama`).
