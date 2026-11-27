---
name: fcode-forms
description: Build a Factorial Code process's input-parameter form shown inside Factorial — enabling it, where it appears (install/settings/uninstall, App page, action dialogs, Factorial One), who may open it and what a public form means for the process behind it, driving behavior from the process return value, multi-step flows, OAuth connect, pairing tables, tables, Set up with One, pre-render and variables, and file uploads. Use when configuring, building, or wiring up a Factorial Code (fcode) form.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Forms

A Factorial Code Form is a process's input-parameter form shown **inside
Factorial**. Each submission starts a process execution with the form data, and
the process's return value decides what the user sees next (a message, inline
errors, the next step). For the schema itself, see `fcode-json-schema`.

Forms are not embedded on external pages: there is no supported script,
React component or modal for third-party sites. A form a user meets lives in
Factorial (table below), or — only with `factorial.form.public` — behind its
own link.

## Gotchas

- **The form *is* the process's `parametersSchema.json`** — there is no separate
  form definition. To change fields/validation/labels, edit the schema.
- **An employee field (`"ui:widget": "employee"`) has a picker only inside
  Factorial** — on a public form it renders a plain select when the field lists
  its choices, and a notice otherwise; see `fcode-json-schema`.
- **The `Forms` flag must be enabled** — in the **Trigger from Factorial**
  section of the process Dashboard, or via `factorial.form.enabled` (legacy:
  `"form": { "enabled": true }`) in the process's `metadata.json` + `fcode push`
  — or the form isn't offered anywhere.
- **A form requires a Factorial identity unless you release it.** A form is
  reached through Factorial, which authenticates the person and injects the
  trusted `fcode.context.factorial`; an anonymous request with the team and
  process slugs gets a `404`. `factorial.form.public` opts out of that for the
  forms that genuinely need it. Either way the process behind the form is the
  guard for *what* the caller may do (below).
- **A form is configured in `factorial.form`**, the Factorial Action block
  (`fcode-factorial-actions`). `form.enabled` still works and is kept mirrored.
  A form offered to users on the App's own page is `factorial.form.appTool`;
  the install / settings / uninstall forms are the processes with those
  reserved slugs (`fcode-factorial-actions`).
- **A schema can't carry executable JavaScript or CSS**, and messages are
  markdown — raw HTML is never rendered. React to input with conditional
  fields, action buttons or steps (`references/advanced.md` "Reacting to user
  input").
- **Never put secrets in the schema or its variables** — they reach the
  browser (see "What a form may trust").
- **Fallback `variables` go in `embedFormOptions.variables`** at the schema
  root — a root-level `variables` key is never read. And a `$ref` to a
  variable that is missing, `null`, `false`, `0` or `""` **removes the node**
  (`references/advanced.md` "Variables").
- **Connecting a third-party account (Slack, GitHub, …) is a schema feature** —
  `"ui:widget": "oauth"` plus an `fcode.oauth` flow whose completion redirects to the
  SDK's callback page. Never ask users to paste an API token into a form when
  the vendor offers OAuth. See "Connect an external account".
- **Mapping many Factorial records to another system is a pairing table**
  (`"ui:field": "pairing"`), not a dropdown per record. A row's proposal goes in
  `x-fcode-suggestion`, never in `default`. See "Pairing tables".
- **Any other grid of values is a table** (`"ui:field": "table"`): fixed rows
  from an `object`, a list the user edits from an `array` of objects. See
  "Tables".
- **A second way to submit is an action button, not a checkbox the user ticks
  before submitting**: a `boolean` with `"ui:field": "button"`, which the
  process receives as `true`. An irreversible action is confirmed with a
  checkbox named in the button's `enabledWhen`, never a second dialog (the
  form is already in one). See "Submit and action buttons".
- **A settings form split with `ui:steps` uses `"navigation": "free"`** (tabs,
  save from any of them); the default is a wizard. See "Multi-step forms".
- **A "Set up with One" button is a `"type": "object"` property drawn by
  `"ui:field": "launch-one"`** that hands the form to One to fill in, never to
  save. Its
  `prompt` and `context` are sent to One: never put secrets in them. See "Set up
  with One".
- **Form text is translated with `fcode.i18n("key")` tokens in the schema**,
  substituted server-side before the schema is served. See `fcode-i18n`.
- **Form submissions run under a request timeout** (about a minute) — keep the
  synchronous process fast, or run long work asynchronously (see below).
- **A fixed wizard across processes is a flow, not a `nextProcessId` chain**:
  declare every step in the entry form's `ui:steps` (`process` steps take
  slugs) and return no `nextProcessId` on the default path. The user can go
  back and re-submit, so every process in a flow must be safe to re-run. See
  "Multi-step forms".
- Drive the UX from the **process return value** (below): there is no
  client-side callback to hook into.

## Enable a form

1. Create the process and define its input parameters (these become the fields).
2. Enable the `Forms` flag — either in the **Trigger from Factorial** section of
   the process Dashboard, or from the CLI workspace in
   `processes/<slug>/metadata.json`, then `fcode push`:

```json
{
  "name": "Contact request",
  "tags": [],
  "factorial": { "enabled": true, "form": { "enabled": true } }
}
```

The form is one entry point of a **Factorial Action** — the same `factorial`
block also holds the button, the Factorial One tool and the backend job
(`fcode-factorial-actions`, field reference in `fcode-cli`), and its
`requiredPolicies` decide who may run it. The legacy top-level
`"form": { "enabled": true }` still works and is kept mirrored with
`factorial.form.enabled`; prefer `factorial` in new work.

Read submitted values in process code like any parameters:
`const { context: { parameters } } = fcode;`

## Where forms appear in Factorial

| Form | Where the user meets it | How it is declared |
|---|---|---|
| **Install** | When a company installs the App, under Integrations | The process slugged `install` |
| **Settings** | When an installed company configures the App | The process slugged `settings` |
| **Uninstall** | When a company removes the App | The process slugged `uninstall` |
| **App tool** | A tab on the App's page in Factorial | `factorial.form.appTool: true` |
| **Action form** | A dialog opened from a button the action places in Factorial (`fcode-ui-triggers`) | `factorial.uiTrigger` next to `factorial.form` |
| **Factorial One** | A card in the One chat, when One opens a lifecycle form | The lifecycle forms above; see "Set up with One" |

The lifecycle slugs are reserved — the slug alone makes a process that form;
the rules, including what never to write, are in `fcode-factorial-actions`. An
`install` or `settings` form is re-opened after the app is configured, so it
should show the **current** values rather than an empty form — the
`preRenderProcess` pattern for that is in `references/advanced.md`.

Factorial draws every form with its own design system (f0); install, uninstall
and action forms open in a dialog, App tools as a tab. Design for that: confirm
an irreversible action with a checkbox in the form, not a second dialog.

## What a form may trust

A form is reached through Factorial, which authenticates the person opening it:
the install / settings / uninstall screens, the forms an App offers on its own
page and the action dialogs all submit through Factorial's backend, which
enforces `requiredPolicies` before anything runs (`fcode-factorial-actions`)
and injects the trusted `fcode.context.factorial`. An anonymous request holding
the team and process slugs gets the same `404` as a form that doesn't exist.

Unless the form sets `factorial.form.public`
(`"factorial": { "enabled": true, "form": { "enabled": true, "public": true } }`).
That flag releases it to anyone holding its link, with no Factorial account,
and then nothing has authenticated the caller at all: the `company_id` and
`access_id` such a submission carries are whatever the request sent, and
`fcode.context.factorial` is absent. Turn it on only for a form meant for
people outside Factorial, and treat everything below as the only protection it
has. It is the **Authentication** field next to the Form switch on the
Dashboard and the legacy `form.authMode` (`NONE` when public), kept mirrored;
a push from a file without `"public": true` makes the form private again
(`fcode-cli`).

That settles *who* is calling. It settles nothing about *what* they submitted,
so the process behind the form is still the guard:

- **Treat every submitted parameter as caller-controlled** — including values
  the form pre-filled or hid (`company_id`, `triggered_from_location`,
  `access_id`). Never authorize on them: read the company and the person from
  `fcode.context.factorial`, and check every value before acting on it.
- **Return nothing the caller shouldn't already have** — the result of a
  submission, and a `preRenderProcess`'s `variables`, reach whoever opened the
  form. A pre-render has no trusted viewer on a public form
  (`references/advanced.md` "Who is opening the form").
- **Keep credentials one-way**: a settings form reports only *whether* a secret
  is set, never its value (`references/advanced.md`). The platform strips the
  `default`/`examples`/`const` of `isSensitive` fields from the schema it
  serves, but a secret put in a label, a markdown block or a `variables` node
  still reaches the caller.
- **Hiding a field is not a permission.** Factorial One reads the schema
  without its `ui` keys, and anyone can submit any values: a rule the form's
  look enforces must be enforced again by the process.

## Handle submission results

A loading overlay shows during execution (its text is
`embedFormOptions.loadingOverlayContent` at the schema root); on success the
form is replaced with a success message, on error an error message. **Drive
what happens from the process return value:**

```js
return { message: "Thanks, **we received your request**." };                // success message (markdown)
return { message: "| Item | Status |\n|---|---|\n| Sync | Done |" };        // GFM tables work too
return { status: 400, body: { formErrors: {                                 // inline validation errors
  fields: { email: "Invalid email." }, global: ["A global error."] } } };
return { status: 400, body: { errorMessage: "**Sync failed** — retry." } }; // error message (markdown)
return { redirect: { url: "https://example.com", timeout: 2000 } };         // redirect after submit
```

Without a `message`, a default confirmation is shown; a thrown error or a
timeout shows a generic error. A `redirect` URL must be `http(s)` (any other
scheme is ignored with a console warning). Install and uninstall forms record
the installation first and then navigate, so a redirecting form that also wants
its `message` read sets a `timeout`.

When a submit fails, with the process's `formErrors` or the form's own
validation, the form brings the first error into view, opening its step first
in a `ui:steps` form: a field's error with its field focused, or, when only the
form as a whole has one (a `global` error), its message. A field error is seen
even on a long form submitted from its foot, so there is no need to repeat it
in a global message. An error inside a table's or a pairing table's cells is
marked there but does not move the view.

**Messages are markdown, not HTML.** `message`, `errorMessage` and a schema's
`markdown.before`/`markdown.after` blocks render as GitHub Flavored Markdown —
tables, headings, links, images, code blocks, task lists. Raw HTML is
**dropped, never rendered** (so nothing authored can execute).

## Keep it fast, or go async

The submission waits for the process to finish, under a request timeout (about a
minute). Heavy work done inline — slow API calls, large exports, multi-record
syncs — will blow the timeout and fail the submit.

**Go async when the work can be slow:**

- **Hand off to another process** — kick off the heavy work with
  `fcode.processes.run("process-identifier", options)` (see `fcode-javascript` /
  `fcode-python`) and return a quick acknowledgement (`message`)
  rather than awaiting it inline.
- **Make the action asynchronous** — with `factorial.awaitResult: false`
  (`fcode-factorial-actions`) Factorial only learns that the execution
  started; the process reports back itself (FactorialClient, a notification).
  It applies to every entry point of the action.

**Stay synchronous only when** the request is genuinely fast, or when data must
flow between steps. For passing data, don't block the submit — instead:

- **`preRenderProcess`** computes server-side `variables` before the form
  renders (see `references/advanced.md`). Opened from Factorial, it can trust
  `fcode.context.factorial` (who is looking) to adapt the form, never its
  parameters; hiding fields is UX only (`references/advanced.md` "Who is
  opening the form").
- **Multi-step forms across processes** — a multi-process flow, or
  `nextProcessId` chaining — carry state forward in `steps` + `variables`
  (below). If the steps are only visual — no work between them — use
  `ui:steps` instead and keep a single process.

## Multi-step forms

Three mechanisms, and picking the right one matters:

| Mechanism | Processes | Executions | Use when |
|---|---|---|---|
| **`ui:steps`** — steps within one form | one | one, after the last step | The split is purely visual; all the logic lives in one process |
| **Multi-process flow** — `process` steps in `ui:steps` | one per form | one per form | An intermediate step must execute something (validate externally, store a credential the next form needs) **and** the steps are known up front |
| **`nextProcessId`** — chaining | one per form | one per form | Each process decides at run time which form comes next |

They compose: a flow's entry form splits into `fields` steps, and a chained
process's form can itself declare `ui:steps`. A wizard whose processes always
return the same fixed `nextProcessId` is a flow written the hard way — declare
the flow instead, so the user gets one navigation and can go back.

**When a form grows long — roughly ten fields or more — split it with
`ui:steps`**, grouping related fields per step with the required ones early.

### Steps within a single form (`ui:steps`)

Declare steps in the schema's root `ui` node, assigning each property to a step.
The SDK shows one step at a time with a "Next" button; only the **last** step's
button submits and starts the (single) execution, which receives every collected
parameter exactly as a one-page form would — process code needs no changes.

```json
"ui": {
  "ui:steps": {
    "config": { "layout": "tabs" },
    "steps": [
      { "title": "About you", "description": "Optional text", "fields": ["name"] },
      { "title": "Details", "fields": ["age", "subscribe"] }
    ]
  }
}
```

- `config` (optional): `layout` — `"tabs"` (default) or `"sidebar"`;
  `navigation` — `"linear"` (default, the wizard described here) or `"free"`
  (tabs of a settings screen, below); `nextLabel` / `backLabel` — button
  labels, default `"Next"` / `"Back"`. An unknown `layout` or `navigation`
  falls back to its default with a console warning. `ui:submitButtonOptions`
  (its `variant` and `enabledWhen` included) applies to the **last** step's
  button; the "Next" buttons keep the primary look and are never held by the
  submit's `enabledWhen`.
- **A step can hold its "Next" until it is answered** with its own
  `enabledWhen` and `disabledReason`, the keys of §Submit and action buttons:
  `{ "title": "What to do", "fields": ["capabilities"], "enabledWhen":
  "capabilities", "disabledReason": "Tick at least one" }`. "Next" stays
  disabled, with the reason as its tooltip, until that root property is `true`
  or a list with at least one item, and the step navigation doesn't go past
  the step either. On the last step it holds the submit too, together with the
  submit's own `enabledWhen`. Use it for a step that needs an answer ("tick at
  least one" under a checkboxes or cards list, an authorisation box) instead of
  `required`, which draws a "*" and lets the user press "Next" into an error.
  Only a `fields` step takes it: a `process` step ignores it with a console
  warning (its form's own submit `enabledWhen` holds it, §Multi-process
  flows), and with `navigation: "free"` there is no "Next" to hold.
- A step can set its own `nextLabel` / `backLabel`, which win over `config`
  for that step (`{ "title": "Credentials", "fields": ["apiKey"], "nextLabel":
  "Verify key" }`). On the last step, `nextLabel` replaces the submit text.
- **A step can hold its "Next" until it is answered** with `enabledWhen` and
  `disabledReason`, the same keys as the submit's: `{ "title": "What to do",
  "fields": ["capabilities"], "enabledWhen": "capabilities", "disabledReason":
  "Tick at least one" }`. "Next" stays disabled — and the step navigation
  doesn't go past it — until that **root** property is `true` or a non-empty
  list (a checkbox list with something ticked). On the last step it holds the
  submit too, together with the submit's own `enabledWhen`. It doesn't apply
  with `"navigation": "free"` (there is no "Next"), and only a step listing
  `fields` takes it: a `process` step ignores it with a console warning, since
  its process's form holds its "Next" with that form's submit `enabledWhen`.
- **Write the button labels in the form's language.** The defaults ("Next",
  "Back", the submit's "Submit") are English whatever the `locale`: set
  `nextLabel`, `backLabel` and `ui:submitButtonOptions.submitText` with
  `fcode.i18n` tokens (`fcode-i18n`) in any form that isn't English-only.
- "Next" validates only the fields on screen (including that step's share of
  `required`), and the view moves to the first error; a Back button and the
  step navigation revisit completed steps without losing input, and unreached
  steps stay disabled.
- **The form's own heading shows once**, above the step navigation: its root
  `title` and `description` (or `ui:title` / `ui:description`), with each
  step's own title and description under it. Don't repeat the form's title in
  every step. `"ui:label": false` in the root `ui` (or in its `ui:options` or
  `ui:globalOptions`) hides it, as on a single-page form.
- Properties not listed in any step are appended to the last one; an unusable
  `ui:steps` falls back to the ordinary one-page form with a console warning.
- Supports root-level `properties` + `required` + `dependencies`; root-level
  `oneOf`/`allOf` compositions are not split into steps.

**Tabs for a settings form (`"navigation": "free"`).** A wizard suits an
install flow; a `settings` form split into sections should not make the admin
press "Next" through every section to change something in the last one. Set
`"config": { "layout": "tabs", "navigation": "free" }` and the steps become
tabs of one form:

- Every step is reachable from the start, in any order. Moving between steps
  validates nothing and keeps what was typed; no step shows as completed.
- The submit button is on every step, with its `ui:submitButtonOptions`
  (`submitText`, `variant`, `enabledWhen`, `disabledReason`). There are no
  "Next" or "Back" buttons, so `nextLabel` / `backLabel` and a step's own
  `enabledWhen` don't apply.
- Submitting validates the whole form, steps never opened included, and sends
  every step's values (the defaults of unopened steps too), as a one-page form
  would. If anything fails nothing is submitted: every step with an error is
    marked in the navigation, the first one opens with its field focused. An
  error that depends on another field (a condition, a check across steps)
  clears as soon as that field is fixed. Field errors the process returns are
  marked and opened the same way.
- The submit's `enabledWhen` may name a property on any step.
- It applies to a form's own steps only: a form that declares `process` steps,
  and a process step's form, ignore it with a console warning (a flow stays
  linear).

### Multi-process flows (`process` steps)

The entry form's `ui:steps` declares every step of the wizard, including the
forms other processes serve: a step with `process: "<slug>"` instead of
`fields` renders that process's whole form. The user sees one navigation for
the whole flow, and the SDK moves from one process to the next on its own.

Worked example — an install form that takes a third-party API key, then maps
teams in a second form whose dropdowns are fetched with that key. The entry
process `vendor-setup` owns the flow:

```json
{
  "title": "Connect Vendor",
  "type": "object",
  "properties": {
    "vendor_api_key": { "title": "Vendor API key", "type": "string", "isSensitive": true }
  },
  "required": ["vendor_api_key"],
  "ui": {
    "ui:steps": {
      "steps": [
        { "title": "Connect", "description": "Enter your API key", "fields": ["vendor_api_key"] },
        { "title": "Map teams", "description": "Map each Vendor team", "process": "vendor-setup-mapping" }
      ]
    }
  }
}
```

```js
// processes/vendor-setup/index.js — runs when "Connect" is submitted
async function main() {
  const { vendor_api_key } = fcode.context.parameters;
  const { VendorClient } = fcode.import("vendor-client");
  await new VendorClient(vendor_api_key).getTeams(); // throws on a bad key
  await fcode.variables.set("VENDOR_API_KEY", vendor_api_key); // overwrite: safe to re-run
  return {}; // no nextProcessId: the flow moves on to "Map teams"
}
```

`vendor-setup-mapping`'s schema sets `preRenderProcess`, and that pre-render
reads `VENDOR_API_KEY` to load the vendor's teams into `#/variables` — it runs
when the "Map teams" form is served, after the key is stored, which a
pre-render on the entry form never could. The full app (Linear, mapping
pre-render, activation, settings, uninstall) is `fcode-examples`
(`references/custom-app-linear.md`).

The rules:

- **The entry form owns the flow.** Only the first form declares the flow's
  steps and its `config` (`layout`, `nextLabel`, `backLabel`), which applies
  throughout. A later form's own `process` steps and `config` are ignored.
  A flow is always linear (`navigation: "free"` is ignored), and each page
  keeps its own form's heading.
- **A `process` step can have sub-steps.** When that process's own form
  declares `ui:steps` with two or more `fields` steps, they show one level
  below the process step in the navigation (`2.1`, `2.2`, …) once the form
  loads. Until then the process step is a single item. They behave like
  in-form steps: only the last one submits, running the process once. Without
  its own `ui:steps`, the process step renders as one page. Use this for a
  long form served by a later process: the flow keeps one navigation, and the
  form keeps its steps when it is opened on its own.
- **`fields` steps first, at least one of them.** They split the entry form
  and are submitted together to the entry process. A `fields` step after a
  `process` step is dropped with a console warning; with no `fields` step at
  all, the whole `ui:steps` is ignored and the form renders as a single page.
- **`process` takes a slug**, like `nextProcessId`. On the marketplace path
  the platform rewrites the slugs into its own action ids — never write ids.
- **Every process in the flow is exposed where the entry form is.** Each step
  loads and submits its own form, so each one needs its form enabled with the
  same access as the entry. A flow started from a UI trigger needs every step
  on the same `uiTrigger.locationId`, which also shows each step as its own
  button there. A step that isn't exposed answers not found in the middle of
  the flow, and the user is left on an error with no way back.
- **No `nextProcessId` on the default path.** After each successful submit the
  SDK moves to the next declared step. Every submit button but the last is
  labelled `config.nextLabel`; the last honours `ui:submitButtonOptions`.
  A page that moves the flow on still submits its form to its process, so its
  "Next" waits for that form's own `ui:submitButtonOptions.enabledWhen` (on
  its last sub-step when it has sub-steps): put a confirmation needed before
  moving on (an authorisation box) in the later form's submit options, not on
  the entry form's `process` step. Per-step `nextLabel` / `backLabel` work on
  `process` steps too; when that form has sub-steps, `nextLabel` goes on the
  sub-step that submits it and `backLabel` on the first one, whose Back
  leaves it.
- **State carries forward as in a chain**: each process receives the earlier
  forms' data and results in `fcode.context.parameters.steps`, and a returned
  `variables` node reaches the following forms.
- **Every process in a flow must be safe to re-run.** The user can go back to
  any step already reached; it reopens filled with what was submitted (a
  process step with sub-steps reopens on its last sub-step), and
  submitting it again re-runs its process and discards every later step's
  `steps` entries, data and variables. Overwrite variables and datastore keys
  rather than appending, and look up webhooks or schedules (or the ids you
  recorded for them) before creating them.
- **Going back drops unsubmitted input, after a confirmation.** Leaving a
  process step for an earlier one, when the user changed a value there
  (sub-steps included), asks first. Moving between a step's own sub-steps never
  asks, and values from schema defaults or a pre-render don't count as
  changes. The wording comes from the entry form's `config`: `leaveTitle`,
  `leaveMessage`, `leaveConfirmLabel`, `leaveCancelLabel` (English defaults).
- **An install flow activates the installation at step 1.** When its entry
  process, `install` (`fcode-factorial-actions`), succeeds, the installation
  becomes `active`, so the later process steps run on an active installation,
  and a user who abandons the flow there leaves it active. Do the setup the App
  can't run without in the entry step.

- **Factorial One fills in each step of a lifecycle flow as it opens.** When
  One opens an `install` or `settings` form in the web chat, each `process`
  step is read only once the user has sent the steps before it, so its
  pre-render sees what they saved (an API key from step 1 that step 2's
  pre-render uses to list options, say). The card hands One the step as it
  read it, so One adds no read of its own; going back to a step reads it
  again, as for any user, and One reuses its proposal if the step did not
  change. One reads the conversation, looks up the Factorial data the step
  needs, and the step opens with its values over the pre-render's; the user
  reviews and sends it. If One cannot propose values, the step opens as the
  pre-render built it. Keep `uninstall` single-process: the App is
  deprovisioned once its uninstall action succeeds, so a flow can't run past
  its entry step. To make a flow fillable:
  - Have each step's process store what the next step needs (a team variable
    or the datastore) and read it back in the next step's pre-render, as the
    Linear setup does with its API key: a pre-render only receives the form's
    context parameters (`fcode.context.parameters`), and the `variables`
    a step returns resolve the next step's `$ref`s without reaching its
    pre-render.
  - Hidden fields are never proposed: they keep their default or the
    pre-render's value.
  - Give options human titles: One picks an option by its meaning and
    proposes its value.
  - Name in a field's title or description the Factorial record it takes
    ("the Factorial legal entity", "the responsible employee"): One looks it
    up by that name when the step opens.
  - Say in the entry process's `agentTool.description` what each step holds
    and what to keep — for example that rows are matched by position, or that
    a proposed list replaces the step's whole list. One reads only the entry
    process's contract, and only with `agentTool.enabled` on; a `process`
    step's own contract is not read. Enabling it also makes `install` or
    `settings` a One tool: the web chat still opens the card, but on clients
    without it (mobile, external assistants) One may run the process from the
    chat once the user approves.

**Overriding the next step.** A process may still return `nextProcessId`:

- naming a **later declared step**, the flow jumps there and the steps in
  between show as skipped;
- naming a process the flow **doesn't declare**, the flow resets: its
  navigation and history go away, and that process's form renders on its own,
  honouring its own `ui:steps`. `steps` and `variables` still carry forward,
  and the console warns. Treat this as an exit from the wizard, not a branch
  inside it.

### Chaining processes (`nextProcessId`)

Each form is its own process, and each process names the next form to show by
returning its **slug** — use this when the route is decided at run time, not
for a fixed sequence (declare a flow for that):

```js
return { nextProcessId: "collect-shipping-address" };
```

The field name is still `nextProcessId` and it accepts a slug or an id — use the
slug, so the same chain works in every workspace.

The SDK then renders the form for `nextProcessId`. Each later step receives all
previous steps' data and results in `fcode.context.parameters` under a `steps`
array. Return a `variables` node alongside `nextProcessId` to pass state forward.
A chain has no shared navigation and no way back: each form replaces the last.

## Connect an external account (OAuth)

A form can make the user authorize a third-party provider before it is
submitted. Declare a `string` or `boolean` property with `"ui:widget": "oauth"`
(an `object` property takes `"ui:field": "oauth"` instead — rjsf never reads
`ui:widget` on objects):

```json
"github_account": {
  "title": "GitHub account",
  "description": "You will be asked to authorize in a new window.",
  "type": "string",
  "ui": {
    "ui:widget": "oauth",
    "ui:options": {
      "authorizationUrl": { "$ref": "#/variables/authorizeUrl" },
      "connectLabel": "Connect GitHub",
      "connectedLabel": { "$ref": "#/variables/connectedLabel" },
      "onComplete": "reload"
    }
  }
}
```

The SDK renders a **Connect** button that opens `authorizationUrl` in a small
popup, waits for the flow to end, closes the popup and fills the field. The
property's `title` / `description` are the message around the button (markdown,
as everywhere; `markdown.before/after` for longer copy).

| `ui:options` | Default | Purpose |
|---|---|---|
| `authorizationUrl` | — | The platform authorization URL returned by `fcode.oauth.start()`. Required; an invalid URL disables the button with a console warning |
| `connectLabel` / `connectedLabel` / `pendingLabel` | `Connect` / `Connected` / `Waiting for authorization…` | Button text before, after and during the flow |
| `onComplete` | `none` | What the form does once connected (below) |
| `popup` | `{ "width": 600, "height": 700 }` | Popup size, centred on the page |
| `closeDelay` | `3000` | Milliseconds the popup stays open after finishing, so the user sees the confirmation |

**The three rules that make it work:**

1. **`authorizationUrl` comes from `fcode.oauth.start()`**, called in a
   `preRenderProcess`, and is injected with `{"$ref": "#/variables/…"}` — a
   plain `{{mustache}}` token is HTML-escaped (`/` → `&#x2F;`, `&` → `&amp;`)
   and the SDK refuses the URL. Pre-render contract in `references/advanced.md`.

   ```js
   const flow = await fcode.oauth.start({
     authorizeUrl: "https://github.com/login/oauth/authorize",
     clientId: fcode.env.GITHUB_CLIENT_ID,
     scope: ["public_repo"],
     onComplete: "github-oauth-callback",
     data: { companyId },
   });
   return { variables: { authorizeUrl: flow.authorizationUrl } };
   ```

   Never build the authorization URL, the `state` or a PKCE pair by hand. The
   platform holds all three, plus the one `redirect_uri` registered with the
   provider — which is the only way this can work at all, since providers
   require a pre-registered URI and every install is its own `deploy-`
   workspace.
2. **The completion process needs no webhook.** The platform invokes
   `onComplete` itself once it has verified and spent the state, so there is no
   endpoint to expose and nothing to authenticate. It arrives with `code`,
   `codeVerifier`, `redirectUri` (replay it in the token exchange — providers
   compare it byte for byte), `data` and `state`, or with `error` /
   `errorDescription` when the provider refused. It exchanges the code, stores
   the tokens server-side (a sensitive variable, or the datastore with the
   encrypted flag — see `fcode-javascript` / `fcode-python`) and ends by
   redirecting the popup to the SDK's callback page:

   ```js
   const { state } = fcode.context.parameters;
   return {
     status: 302,
     headers: {
       Location: "https://code.factorialhr.com/sdk/oauth-callback.html"
         + "?status=success&value=" + encodeURIComponent(login)
         + "&state=" + encodeURIComponent(state),
     },
   };
   ```

   That page tells the form how it went — the form only trusts a message from
   the very window it opened — then closes itself. Query parameters: `status`
   (`success`; anything else counts as an error), `value` (becomes the field
   value), `state` (**required**, echoed from the callback parameters), `message`
   (shown under the button on error), plus any extra parameter an `object` field
   should receive.

   Echo `state` on **every** exit, the error ones included. It is what binds the
   outcome to the popup the form opened, and a completion that omits it is
   discarded when it arrives over the fallback channel — the COOP-severed case
   `onComplete: "reload"` exists to recover. The flow then ends as cancelled even
   though the token exchange succeeded and the token was stored.
3. **`value` is an opaque handle** (an account login, a connection id) — never
   a token: it reaches the browser and travels in the submission. The connected
   state is a signal for the user, not a proof: the process receiving the
   submission must verify the connection on its side.

**Field value.** `boolean` → `true`; `string` → the `value` parameter;
`object` → every parameter except `status` / `message`. Empty until connected,
so marking the field `required` keeps the form from being submitted before the
account is connected. Closing the popup early counts as cancelled and re-enables
the button. An error shows its `message` under the button **and reloads the
form definition, whatever `onComplete` says**: the callback has run and may have
consumed the one-time `state` in the authorization URL, so only a fresh
`preRenderProcess` run can put a working URL behind the button. Typed values and
the message survive the reload, nothing is submitted, and the user simply
retries. A reload that fails leaves the form as it was.

**A `default` renders the button connected.** The button is in its connected
state whenever the field has a value, so a `preRenderProcess` that finds an
existing connection returns the property with a `default` set to the connection
handle — the button renders as connected (disabled) and the handle travels with
the submission like any other default. This is how a returning user, a
connection finished after the popup was closed, or an `onComplete: "reload"`
lands on the connected view; a value arriving this way also clears an earlier
error message.

**`onComplete`** — what happens once connected:

- `none` — the button shows its connected state; the user submits as usual and
  the process receives the value with the other parameters.
- `submit` — the form is submitted immediately, so the return value's
  `message` / `redirect` / `nextProcessId` apply right away. In a `ui:steps`
  form this advances to the next step.
- `reload` — the form fetches its definition again, re-running the
  `preRenderProcess`, so the server can check the connection and render the
  connected state (different copy, more fields, a `default` that turns the
  button into "Connected as …"). Values typed into other fields are kept.
  Because the server is the source of truth here, the form also reloads when
  the popup is closed without reaching the callback page — the only mode that
  reloads on cancel, since the callback never ran and the URL is still good.

Limits worth knowing: the popup is opened from the click itself, so default
popup blockers let it through — a browser set to block *all* pop-ups gets a
message under the button instead. A provider served with a
`Cross-Origin-Opener-Policy` header can sever the link with the popup; inside
Factorial the completion still arrives, and `onComplete: "reload"` is the
safest choice for such providers.

A complete sample — pre-render starting the flow, the completion process, the form
process verifying the connection — is in `fcode-examples`
(`references/oauth-connect.md`).

## Submit and action buttons

**The submit button** is set in the root `ui` node's `ui:submitButtonOptions`:
`submitText`, and `variant`, its look — `default` (the primary button, used
when absent), `outline`, `neutral`, `critical` (a destructive action, in red)
or `ghost`. An unknown variant logs a warning and keeps `default`.

**Action buttons** give a form more than one way to submit it: **Check
connection** next to **Save**, **Skip for now** next to **Continue**,
**Disconnect** on a settings form. Declare a `boolean` property with
`"ui:field": "button"` for each one. It renders as a button where the property
sits among the fields, and a click sets that property to `true` and submits the
form:

```json
"checkConnection": {
  "title": "Check connection",
  "description": "Tries the credentials without saving them.",
  "type": "boolean",
  "ui": { "ui:field": "button", "ui:options": { "variant": "outline" } }
}
```

Clicking it submits every value plus `"checkConnection": true`; **Save** submits
the same values without the key. The process branches on it (test the
credentials and return a `message`, or store them).

- **Label** — the property's `ui:title`, then `title`, then its name. The
  `description` (markdown) shows under the button.
- **Look** — `ui:options.variant`, the same values as the submit's; `outline`
  when absent (the usual secondary action). An unknown value logs a warning and
  keeps `outline`.
- **Validation runs as for the submit button**: with a required field empty
  nothing is submitted. So an action can't skip validation; a "Back to the
  list" action only works on a form with nothing left `required`.
- **The `true` only travels in the submission the click triggered.** After
  that submit the property goes back to what it held, so a later **Save**
  doesn't carry it.
- **Enter** in a text field always submits through the submit button, never
  an action button.
- **In a `ui:steps` form, put action buttons on the last step.** On an earlier
  step a click validates that step and moves on, like "Next", and the `true`
  stays with the values submitted at the end.

**Enable a button only after confirming.** Install, uninstall and action
forms already open in a dialog inside Factorial, so an irreversible action is
not confirmed with a second dialog on top: ask for a checkbox ("I understand
that disconnecting stops the sync") and keep the button disabled until it is
ticked. Name that root `boolean` property in `enabledWhen`, and say why the
button is disabled in `disabledReason` (its tooltip):

```json
{
  "type": "object",
  "properties": {
    "confirm": { "title": "I understand that disconnecting stops the sync", "type": "boolean" }
  },
  "required": ["confirm"],
  "ui": {
    "ui:submitButtonOptions": {
      "submitText": "Disconnect",
      "variant": "critical",
      "enabledWhen": "confirm",
      "disabledReason": "Tick the box to confirm"
    }
  }
}
```

- The button is enabled only while the property is exactly `true`, or a list
  with at least one item (a `checkboxes` or cards list where something is
  ticked: "pick at least one" without `required`); anything else (empty,
  `false`, a string, an empty list) keeps it disabled. It follows the value
  live (untick and it disables again). While it is disabled, Enter doesn't
  submit either.
- `enabledWhen` names a property at the **root** of the form (no `"a.b"`
  paths). `disabledReason` is optional and shows only while the button is
  disabled (on hover, and to screen readers). **Mistakes fail closed:** a
  non-string `enabledWhen`, or one naming a property the form doesn't have (a
  typo), keeps the button disabled for good and logs a console error — check
  the name when a button never enables.
- An action button takes the same two keys in its `ui:options`
  (`{ "variant": "critical", "enabledWhen": "confirm", "disabledReason": "…" }`);
  each button follows its own rule.
- In a `ui:steps` form the submit's `enabledWhen` reads the whole form, so
  the confirmation may sit on any step (the last one, next to the button,
  reads best); "Next" is never disabled by it. A step holds its own "Next"
  with its own `enabledWhen` (§Steps within a single form), and in a
  multi-process flow a later form's submit `enabledWhen` holds the "Next"
  that submits it.
- **It only changes the form on screen.** The process still checks the value
  (`if (confirm !== true) …`) before doing anything irreversible: a submission
  can reach it without going through the form.

With "Set up with One", One never clicks an action button nor ticks the
property an `enabledWhen` names (see "Set up with One").

## Pairing tables

Integrations map Factorial records to their counterparts elsewhere: expense
accounts to a NetSuite chart of accounts, payroll concepts to DATEV wage types,
absence types to an official code list. One dropdown per record turns hundreds
of records into a form nobody can review. Mark the section
`"ui:field": "pairing"` and Factorial draws it as one table: per row the
Factorial record, a searchable picker of the other system's records (a text
input where they can't be listed, see "Typed rows" below), and a
status — **Confirmed** (has a value, including an explicit "Not mapped"),
**Suggested** (the picker shows the proposal, marked as not chosen yet; the
app's reason on hover), **No longer available** (the stored pair is no longer a
choice) or **Needs a choice** — plus, above the table, **Accept N suggestions**
and a **Needs a choice (N)** filter that narrows the list to the open rows (a
row paired meanwhile stays listed until the filter is turned off), and
**Accept** on each row showing a suggestion.

```json
"accounts": {
  "type": "object",
  "title": "Expense accounts",
  "ui": {
    "ui:field": "pairing",
    "ui:options": {
      "columns": ["Factorial", "NetSuite"],
      "inUseNote": "Only the accounts your expense categories use are listed.",
      "labels": { "addRow": "Add account" }
    }
  },
  "x-fcode-choices": [
    { "const": "", "title": "Not mapped" },
    { "const": "6100", "title": "6100 · Travel" },
    { "const": "6200", "title": "6200 · Meals" }
  ],
  "properties": {
    "acc_12": { "type": "string", "title": "Travel", "default": "6100" },
    "acc_13": {
      "type": "string",
      "title": "Meals",
      "x-fcode-suggestion": { "value": "6200", "reason": "Same name" },
      "x-fcode-in-use": true
    }
  }
}
```

The contract:

- **One property per row.** The property name is the key the submission
  carries (use the Factorial record's id, prefixed so it is a valid key:
  `acc_12`), `title` is the Factorial side. A property that is not a `string`
  is left out of the table with a console warning. The section may be
  marked on its own `ui` or from the root `ui` / `uiSchema` map by name.
- **Choices once per section.** `x-fcode-choices` (a list of
  `{ const, title }`) on the section is used by every row without its own
  `oneOf` / `enum`. Repeating a long list on every row makes the schema grow
  with rows × records; a row that needs a different list keeps its own.
- **`default` is the stored pair** and shows as Confirmed. A stored pair that
  is no longer among the choices (deleted or deactivated on the other side) is
  set aside: the row reads **No longer available: <value>**, needs a new
  choice, and the old value is not submitted.
- **`x-fcode-suggestion: { value, reason }` is a proposal**, never submitted
  until the user accepts it or picks another record: saving without accepting
  leaves the row unpaired. Clearing a row dismisses its suggestion. A `value`
  that is not one of the row's choices is ignored (on a typed row any text
  goes). Leave its `source` out: your
  app's suggestions are `"app"`, and `"one"` marks One's (see "Set up with One").
- **Typed rows, when the other system can't be listed.** A section that
  sets `"ui:options": { "freeText": true }` types its rows: there, a string row
  with no `oneOf`, no `enum` and no section `x-fcode-choices` is a text input, the admin types
  the other system's value into a text input, and the submission carries what
  was typed. Without `freeText`, a row with no choices is left out of the table
  (with a console warning), so a pre-render that fails to fetch the vendor's list
  gives an empty table, not silent free text. Opt in only when the vendor has no
  API to list the records or the app has no access to it — don't invent a fake
  choice list. It reads Needs a
  value (not Needs a choice) until something is typed, then Confirmed; blanking
  it clears the row. When every row of a section is typed, the filter reads
  Needs a value (N).
  Suggestions and `default` work as on any row. Typed rows have no "Not
  mapped" and never read No longer available. Since anything can be typed,
  constrain the row (`pattern`, `maxLength`: the error shows on that row) and
  check the values again in the process. In an opted-in section, rows with
  choices stay selects, so typed rows and rows with choices mix. `freeText`
  must be a boolean; anything else warns and counts as off.

  ```json
  "codes": {
    "type": "object",
    "title": "Wage type codes",
    "ui": { "ui:field": "pairing", "ui:options": { "freeText": true, "columns": ["Factorial", "Payroll code"] } },
    "properties": {
      "concept_4": { "type": "string", "title": "Base salary", "pattern": "^[0-9]{4}$", "default": "1000" },
      "concept_7": { "type": "string", "title": "Overtime", "pattern": "^[0-9]{4}$" }
    }
  }
  ```
- **`x-fcode-in-use: true`** marks Factorial records in use. While any row is
  marked, the table lists only those plus rows paired or suggested, and
  **Add row** brings in any other row. Mark them on long lists: each edit
  re-renders every listed row, and a few hundred listed rows make every change
  lag noticeably.
- **"Not mapped" is an explicit choice.** A `""` choice is offered in the
  picker under its own title; picking it submits `""`. A stored `""` is
  Confirmed and a suggestion never replaces it, so only send `"default": ""`
  when the admin chose "not mapped" (not as a blank for every row). This holds
  only when the row's choices include a `""` option: without one, a stored `""`
  is dropped, the row reads Needs a choice, and the next save leaves the key out. Clearing a
  row (✕) leaves its key out of the submission.
- **Submission** is the same flat `{ "acc_12": "6100", … }` object a section of
  ordinary dropdowns submits, so the handler reads it as it always did; the
  process must still check every value against the other system's records
  (parameters are untrusted, §What a form may trust).
- **`ui:options`** (all optional): `columns` (two headers), `inUseNote`, and
  `labels` overriding the table text — `status`, `confirmed`, `suggested`,
  `suggestedByOne`, `oneSuggests` (`{value}`), `use`, `needsChoice`, `needsValue`, `unavailable` (`{value}`), `accept`,
  `acceptAll` (`{count}`),
  `summary` (`{confirmed}`, `{toReview}`), `placeholder`, `addRow`,
  `needsChoiceFilter` (`{count}`), `needsValueFilter` (`{count}`), `empty`.
  Translate them with `fcode.i18n`
  tokens (`fcode-i18n`). A bad option falls back to its default with a console
  warning.
- **Validation** works per row: a row listed in the section's `required` must be
  paired, and the error shows on that row, which is always listed, even when
  the in-use list or the filter would leave it out.

Build the section in a `preRenderProcess` (`references/advanced.md`): rows from
the Factorial records, `x-fcode-choices` from the other system, `default` from
what the app stored, `x-fcode-suggestion` from the app's own matching (same
number, same name…), `x-fcode-in-use` from what Factorial actually uses. Inject
the whole section as one variable
(`"accounts": { "$ref": "#/variables/accountsField" }`).

A complete app — pre-render building the section, settings process
storing the pairs — is in `fcode-examples` (`references/pairing-table.md`).

## Tables

When a section is a grid of values (one line per expense category, a list of
invoice lines), mark it `"ui:field": "table"` and Factorial draws it as one
table instead of a stack of fields. The schema decides the columns, the rows
and what each cell holds; the table adds only cell editing, a **select all**
checkbox over a yes/no column and, on a list, adding, removing and reordering
rows. For mapping records to another system, with suggestions and statuses,
use a pairing table instead.

**Fixed rows — an `object` with one `object` property per row.** The row
property's `title` labels the row, its properties are the row's cells. Use it
when the app knows the rows (build them in a `preRenderProcess`).

```json
"categories": {
  "type": "object",
  "title": "Expense categories",
  "ui": {
    "ui:field": "table",
    "ui:options": {
      "rowHeader": "Category",
      "selectAll": "billable",
      "columns": [
        { "key": "account", "title": "Account" },
        { "key": "tax", "title": "Tax", "width": 180 },
        { "key": "billable", "title": "Billable" },
        { "key": "state", "title": "State", "width": 140 }
      ]
    }
  },
  "x-fcode-choices": {
    "tax": [ { "const": "vat21", "title": "VAT 21%" }, { "const": "vat10", "title": "VAT 10%" } ]
  },
  "properties": {
    "cat_12": {
      "type": "object",
      "title": "Travel",
      "required": ["account"],
      "properties": {
        "account": { "type": "string", "default": "6100" },
        "tax": { "type": "string" },
        "billable": { "type": "boolean" },
        "state": { "type": "string", "default": "Synced", "ui": { "ui:widget": "badge", "ui:options": { "variant": "positive" } } }
      }
    }
  }
}
```

Once filled in, that submits `{ "categories": { "cat_12": { "account": "6100",
"tax": "vat21", "billable": true, "state": "Synced" } } }`, one entry per row.

**A list — an `array` of objects.** Every row shares the `items` schema and the
user adds and removes rows (`minItems` / `maxItems` respected); a new row starts
with each cell's `default`. It submits an array of row objects.

```json
"lines": {
  "type": "array",
  "title": "Invoice lines",
  "minItems": 1,
  "ui": { "ui:field": "table", "ui:options": { "orderable": true, "labels": { "addRow": "Add line" } } },
  "items": {
    "type": "object",
    "required": ["concept"],
    "properties": {
      "concept": { "type": "string", "title": "Concept" },
      "amount": { "type": "number", "title": "Amount", "minimum": 0, "ui": { "ui:options": { "units": "€", "maxDecimals": 2 } } }
    }
  }
}
```

**A cell is whatever the row's property is**, so with fixed rows two rows can
hold different things in one column, and a row without the property leaves the
cell empty:

| The property is… | The cell is… |
|---|---|
| `string` | Text input; `format` `email` / `uri`, or `ui:options.inputType` (`text`, `email`, `url`, `tel`), sets the input type |
| `string` with `"format": "date"` | Date picker; the value is a day (`2026-11-20`) |
| `number` / `integer` | Number input; `minimum` / `maximum`, and `ui:options` `units`, `unitsPosition` (`"before"` / `"after"`, default after), `maxDecimals` (an `integer` takes none) |
| `boolean` | A Yes / No choice (`display: "switch"` doesn't apply in a table) |
| `string` with `oneOf` of `{ const, title }` or `enum` | Searchable select; nothing chosen until the user picks, unless there is a `default` |
| `array` whose `items` are such a string | Multiselect; the value is the list of chosen values |
| any of these with `"readOnly": true` | The value as plain text (a choice's title for a select) |
| any of these with `"ui:widget": "badge"` | A coloured, non-editable badge: `ui:options.variant` `neutral` (default), `info`, `positive`, `warning` or `critical`; on a list, where rows share a schema, colour by value with `"variants": { "draft": "warning", "sent": "positive" }` |

The contract:

- **Read-only and badge cells show a value the app set** (a `default`, usually
  from the pre-render), and it is submitted back as it came.
- **A property's `description` becomes a hint** on its cell (an info icon).
- **One list of choices per column.** `x-fcode-choices` on the section is keyed
  by **column** (unlike a pairing table's single list): every cell of that
  column with no `oneOf` / `enum` of its own uses it, as a select (a string) or
  a multiselect (an array). A cell can still carry its own list.
- **Select all.** `ui:options.selectAll` names a yes/no column; its header gets
  a checkbox that sets every row's cell to `true` or `false`. The submission is
  still each row's own value. A column the table doesn't have, or one with no
  yes/no cell, is not offered and logs a console warning.
- **Validation and clearing.** A property in the row's `required` must be
  filled, and errors show on the cell. Clearing a cell leaves its key out of
  the submission.
- **On fixed rows, keep a column to one kind of input.** A text cell's input
  type and a number cell's decimals, `minimum` and `maximum` are set per column
  from the first row that has such a cell; `units` can differ per row.
- **`ui:options`** (all optional): `columns` (property names or
  `{ key, title, width }`, `width` in pixels; default every property the rows
  have, titled by the property's `title`), `rowHeader` (fixed rows: header of a
  first column with each row's `title`), `addable` / `removable` (lists,
  default `true`), `orderable` (lists, default `false`), `selectAll`, and
  `labels` overriding the table's own text — `addRow`, `removeRow`,
  `selectAll`, `yes`, `no`, `placeholder`, `empty`. Those texts already follow
  the form's `locale` (Spanish for `es`, English otherwise, `fcode-i18n`), so
  set a label only for other wording, with `fcode.i18n` tokens; a label set
  there stays as written. A bad option falls back to its default with a console
  warning; a fixed row that is not an object is left out, and a cell that is an
  object or an array shows empty.
- **Size.** Each edit re-renders every row: tens of rows, not thousands.


## Set up with One

A long setup, or a pairing table with hundreds of rows, can be handed to One
(Factorial's AI assistant) from the form itself. Add a `"type": "object"`
property drawn by `"ui:field": "launch-one"`: it shows only the button, submits
nothing, and inside Factorial opens a new conversation with One about this form.

```json
"setupWithOne": {
  "type": "object",
  "ui": {
    "ui:field": "launch-one",
    "ui:options": {
      "label": "Set up with One",
      "prompt": "Set up the NetSuite integration",
      "context": "The subsidiary chosen so far: \"{{subsidiary}}\". Use the subsidiary of the company's main legal entity. Match accounts by number first, then by meaning; never pick a parent account."
    }
  }
}
```

- **Add it only where One can work the values out**: records to match by
  meaning (a mapping, a pairing table), values in the company's own data or
  documents (a policy, the handbook), or a setup the `context` can describe in
  words. Leave it off a form only the admin can answer (credentials, choices
  that are theirs to make) and off a short form that is quicker to fill in by
  hand: a button that leaves most fields empty costs the admin more than it
  saves.
- **When the values live in another system, have One ask for them.** A form may
  need identifiers or codes that only another tool holds, so One cannot find
  them in Factorial. If that tool can export them (a CSV) or show them on one
  screen (a screenshot), the `context` tells One to ask the admin for exactly
  that export, then fill the form in from it: name the export or screen as the
  other tool calls it, and say which column or label goes to which field. One
  asks in the conversation, reads the file the admin attaches and proposes the
  values. What it read reaches the form as part of a short note (about 8,000
  characters), so this suits dozens of rows, not thousands: for those, have the
  app read the other system itself and fill a pairing table.

  ```json
  "context": "The NetSuite internal ids of the subsidiaries are not in Factorial. Ask the admin for the CSV export of Setup > Company > Subsidiaries in NetSuite (or a screenshot of that list), then fill in each subsidiary's internal id from its Internal ID column, matching rows by name."
  ```
- **`label`** (required) is the button text and **`prompt`** (required) the
  short message shown in the conversation as the user's. **`context`** is
  longer instructions sent along unseen: how this app is set up, what to match
  by, what to leave alone. **`section`** names the field or pairing section One
  works on; without it, the whole form. A form may have several buttons, for
  example one per pairing table with its own `section`, placed where it should
  appear.
- **`collapse: true`** (off by default, and only with `section`) moves the
  button into that section, right after its title and description, and starts
  the section's content collapsed under a "Show the full form" toggle beside it.
  Collapsed fields stay in the form: they keep their values, validate and are
  saved, and one with an error opens. Secret, file and hidden fields never
  collapse, nor do action buttons and the confirmation an `enabledWhen` names:
  they stay in sight. Change the toggle's text with `labels.showForm` and
  `labels.hideForm`. Without a `section`, `collapse` is ignored with a console
  warning; where the button isn't shown, nothing collapses.
- **`{{field}}` and `{{variable}}`** in `prompt` and `context` are filled in
  from the form's current values when the user clicks. Only plain names, no
  Mustache sections. Secret, file and hidden fields (and anything holding one),
  and variables that only feed secret fields, are filled in as empty.
- **What One does.** It reads the form as it stands, with the user's edits,
  and proposes: values for the fields the user has not changed, and a
  suggestion with a reason for each open pairing row, marked **Suggested by
  One**. On a confirmed row it disagrees with, the pair stays and the row offers
  **One suggests X** with **Use**. It never fills a secret, file or hidden
  field, never proposes a value for an action button or for the property an
  `enabledWhen` names (only the admin confirms), never touches a field the
  user changed or a row they dismissed, and
  **never saves the form or runs the process**: the user reviews and saves. After
  the save, One reports the result in the same conversation.
- **What One knows of your app** is the button's texts and the form's titles,
  descriptions and choices. Write `context` for a person setting the app up for
  the first time. The user's company data One looks up itself.
- **Never put secrets in `prompt` or `context`**: they go to One. Do not ask One
  to choose notification recipients or to tick confirmations for the user; it
  won't.
- **Where the button shows** is the host's decision: on forms opened from
  Factorial (Integrations, the App's page, trigger forms) for users and
  companies with One. It renders nothing in the form card One itself shows in
  a conversation, on a public form, or for a company without One, so the
  schema stays valid everywhere.

A complete sample, with a button for a pairing table, is in `fcode-examples` (`references/pairing-table.md`).

## Automatic file uploads

A file field is `"type": "string"` with a `"ui": { "ui:widget": "file" }` key
**inside the property**:

```json
{
  "properties": {
    "inputFile": { "type": "string", "ui": { "ui:widget": "file" } }
  }
}
```

(A root-level `ui` map keyed by field name — rjsf's `uiSchema` convention — is
also merged, but per-property `ui` is the documented form; the root `ui` is
mainly for form-level options like `ui:submitButtonOptions`. For secret inputs
use `"isSensitive": true` on the property — it renders a password widget
automatically, while `"ui:widget": "password"` alone only hides the value and
the CLI warns about it on push; see `fcode-json-schema`.)

On submit the file is **uploaded to Storage before the process starts**, and the
parameter arrives as an `fcode.storage://…` reference (an **array** if multiple
files allowed). Strip the prefix to download:

```js
const { context: { parameters } } = fcode;
const stream = await fcode.storage.download(
  parameters.inputFile.replace("fcode.storage://", "")
);
```

Uploaded files count toward storage limits (1,000 files, 50 MB each —
`fcode-core-concepts` › Plans & limits) — delete them at the end of the process
if only needed transiently.

## Advanced

For what you control of a form's look, variables (`embedFormOptions.variables`,
`$ref` and Mustache, precedence, the empty-value rule), the server-side
pre-render (who is opening the form, pre-filling current values into
install/settings forms) and reacting to user input without client code, read
`references/advanced.md`.
