---
name: fcode-forms
description: Embed a Factorial Code process's input-parameter form on a webpage — the three embed methods (data attributes, Fcode.initForm, the FcodeForm React component), version pinning to the stable alias, access restriction (authMode), driving behavior from the process return value, styling and themes (including the f0 theme inside Factorial), multi-step flows, pre-filling current values, and file uploads. Use when embedding, configuring, styling, restricting, version-pinning, pre-filling, or wiring up a Factorial Code (fcode) form.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Forms

A Factorial Code Form embeds a process's input-parameter form on any webpage.
Each submission starts a process execution with the form data, and the result is
handled in-page (messages, redirects, callbacks). For the schema itself, see
`fcode-json-schema`.

## Gotchas

- **The form *is* the process's `parametersSchema.json`** — there is no separate
  form definition. To change fields/validation/labels, edit the schema, **not**
  the embed code.
- **`team` and `process`/`processId` are both mandatory** on every embed, and
  both take **slugs** — not the per-workspace UUIDs.
- **Always pin the embed to the `stable` alias**
  (`data-fcode-form-process-version="stable"` / `processVersion: "stable"`).
  An unpinned form runs the current version, so every `fcode push` changes it
  immediately. Details below.
- **An unknown version or alias doesn't fail the form** — it silently runs the
  current version (see "Pin the form to a version").
- **The `Forms` flag must be enabled** — on the process Dashboard, or via
  `"form": { "enabled": true }` in the process's `metadata.json` + `fcode push`
  — or the embed won't render.
- **A new form requires a Factorial user by default** (`authMode: FACTORIAL`).
  An embed on a public page needs `authMode: NONE`, or every request gets a
  `401` (below).
- **A schema can't carry executable JavaScript.** `embedFormOptions.onChange`
  and field `transformFn` were removed, and messages are markdown — raw HTML is
  never rendered. Client-side behaviour lives in the embedding page.
- **Never put secrets in embed code or `options`** — they run in the browser.
- **Form text is translated with `fcode.i18n("key")` tokens in the schema**,
  substituted server-side before the schema is served. See `fcode-i18n`.
- **Form submissions run under a request timeout** (about a minute) — keep the
  synchronous process fast, or run long work asynchronously (see below).
- Prefer driving UX from the **process return value** (below); reserve
  `onSuccess`/`onError` for client-only logic.

## Enable a form

1. Create the process and define its input parameters (these become the fields).
2. Enable the `Forms` flag — either on the process Dashboard, or from the CLI
   workspace in `processes/<slug>/metadata.json`, then `fcode push`:

```json
{
  "name": "Contact request",
  "tags": [],
  "form": { "enabled": true }
}
```

For marketplace app processes, `form` also takes an optional
`"appRole"` (`INSTALL` | `SETTINGS` | `USER_FACING_FORM` | `UNINSTALL`) marking
the process's role in the app. Field reference in `fcode-cli`.

An `INSTALL` or `SETTINGS` form is re-opened after the app is configured, so it
should show the **current** values rather than an empty form — the
`preRenderProcess` pattern for that is in `references/advanced.md`.

Read submitted values in process code like any parameters:
`const { context: { parameters } } = fcode;`

## Restrict who can open the form

The `Authentication` field next to the `Forms` flag (`form.authMode` in
`metadata.json`) decides who may read the form schema **and** submit it:

| `authMode` | Who gets in |
|---|---|
| `FACTORIAL` | Only Factorial users of the company that installed the app. Every request must carry a Factorial-issued user token in the `Fcode-Factorial-Token` header, and that token's company must own the workspace. Anything else gets a `401` |
| `NONE` | Anyone who knows the form URL can open and submit it |

- Forms embedded **inside Factorial** (the marketplace `INSTALL` / `SETTINGS` /
  `USER_FACING_FORM` / `UNINSTALL` screens) send the token for you — this is what
  `FACTORIAL` is for.
- A protected form is still openable from the **playground link** on the process
  Dashboard: the playground sends the developer's own Factorial Code token as
  `Fcode-Platform-Token` and access is granted through workspace membership.

Field encoding rules (`authMode` omitted when `NONE`, lifting protection needs
an explicit `"authMode": "NONE"`) and the full reference in `fcode-cli`.

## Embed a form

Two mandatory inputs, both **slugs**, plus the version pin you should always
add:

- **`fcode-team-slug`** — from `https://code.factorialhr.com/platform/<fcode-team-slug>`
- **`fcode-process-slug`** — the **Slug** field on the process Dashboard (e.g.
  `send-welcome-email`)
- **process version** — pin it to the `stable` alias (next section)

**Use the slug, not the process ID.** IDs are per-workspace UUIDs, so an
id-based embed breaks when the snippet moves between workspaces; slugs survive
(existing id-based embeds keep working, and the React prop is still named
`processId` — feed it a slug). Slugs are editable with no redirect for the old
value, so settle them before handing out embed code — renaming one breaks every
embed already pasted into a page.

Load the SDK once (needed for the data-attribute and `Fcode.initForm` methods):

```html
<script defer src="https://code.factorialhr.com/sdk/forms.js"></script>
```

**Method 1 — data attributes** (SDK replaces the element):

```html
<div
  data-fcode-form-team="<fcode-team-slug>"
  data-fcode-form-process="<fcode-process-slug>"
  data-fcode-form-process-version="stable"
></div>
```

**Method 2 — `Fcode.initForm`** (selector or DOM element):

```html
<div id="my-fcode-form"></div>
<script>
  Fcode.initForm("#my-fcode-form", {
    team: "<fcode-team-slug>",
    process: "<fcode-process-slug>",
    processVersion: "stable",
  });
</script>
```

**Method 3 — `FcodeForm` React component** (React 17/18; install
`@factorialco/fcode-react-forms`):

```jsx
import FcodeForm from "@factorialco/fcode-react-forms";

const MyComponent = () => (
  <FcodeForm
    team={"<fcode-team-slug>"}
    processId={"<fcode-process-slug>"}
    processVersion={"stable"}
  />
);
```

In SSR frameworks (e.g. Next.js), import it dynamically with `ssr: false`.

## Pin the form to a version

The version pin (`data-fcode-form-process-version` / `processVersion`) takes a
published version tag (`v1.0.0`) or an alias — use `stable`, so releases and
rollbacks happen by moving the alias, never by editing the embedded page
(alias model in `fcode-core-concepts`; release commands in `fcode-cli`). The
pin applies to **both** requests the form makes — loading the definition and
submitting it — and the process Dashboard writes it for you: pick a version or
alias in the selector next to the embed code and copy the generated snippet.

**An unknown version or alias silently runs the current version** — the
platform only logs a server-side warning, same as webhooks (see `fcode-cli`).
A typo in the attribute is invisible: check the execution's version when a
submission behaves unexpectedly. Direct calls to the form endpoints also accept
the `version_tag` query parameter documented in `fcode-cli`.

## Handle submission results

Default: a loading overlay shows during execution; on success the form is
replaced with a success message, on error an error message.

**Callbacks** (same shape across methods):

```js
Fcode.initForm("#my-fcode-form", {
  team: "<fcode-team-slug>",
  process: "<fcode-process-slug>",
  onSuccess: (formId, processExecutionResult, formSubmittedData) => {},
  onError: (formId, error, formSubmittedData) => {},
});
```

With data attributes, point to global functions via
`data-fcode-form-on-success="HANDLER_NAME"`,
`data-fcode-form-on-next-step="..."` and `data-fcode-form-on-error="..."`.

**Drive behavior from the process return value** (no client code needed):

```js
return { message: "Thanks, **we received your request**." };                // success message (markdown)
return { message: "| Item | Status |\n|---|---|\n| Sync | Done |" };        // GFM tables work too
return { status: 400, body: { formErrors: {                                 // inline validation errors
  fields: { email: "Invalid email." }, global: ["A global error."] } } };
return { status: 400, body: { errorMessage: "**Sync failed** — retry." } }; // error message (markdown)
return { redirect: { url: "https://example.com", timeout: 2000 } };         // redirect after submit
```

**Messages are markdown, not HTML.** `message`, `errorMessage` and a schema's
`markdown.before`/`markdown.after` blocks render as GitHub Flavored Markdown —
tables, headings, links, images, code blocks, task lists. Raw HTML is
**dropped, never rendered** (so nothing authored can execute); put behaviour in
the success / next-step / error callbacks instead.

## Keep it fast, or go async

The submission waits for the process to finish, under a request timeout (about a
minute). Heavy work done inline — slow API calls, large exports, multi-record
syncs — will blow the timeout and fail the submit.

**Go async when the work can be slow:**

- **Embed `async: true`** — the submission returns `201` + an execution ID
  immediately instead of waiting for the result (see `references/advanced.md`).
- **Hand off to another process** — kick off the heavy work with
  `fcode.processes.run("process-identifier", options)` (see `fcode-javascript` /
  `fcode-python`) and return a quick acknowledgement (`message`/`redirect`)
  rather than awaiting it inline.

**Stay synchronous only when** the request is genuinely fast, or when data must
flow between steps. For passing data, don't block the submit — instead:

- **`preRenderProcess`** computes server-side `variables` before the form
  renders (see `references/advanced.md`).
- **Chained multi-step forms** carry state forward via `nextProcessId` +
  `variables` (below). If the steps are only visual — no work between them —
  use `ui:steps` instead and keep a single process.

## Multi-step forms

Two mechanisms, and picking the right one matters:

| Mechanism | Processes | Executions | Use when |
|---|---|---|---|
| **`ui:steps`** — steps within one form | one | one, after the last step | The split is purely visual; all the logic lives in one process |
| **`nextProcessId`** — chaining | one per step | one per step | An intermediate step must actually execute something (validate externally, compute `variables` for the next form) |

They compose: a chained process's form can itself declare `ui:steps`.

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
  `nextLabel` / `backLabel` — button labels, default `"Next"` / `"Back"`.
  `ui:submitButtonOptions` applies to the **last** step's button.
- "Next" validates only the fields on screen (including that step's share of
  `required`); a Back button and the step navigation revisit completed steps
  without losing input, and unreached steps stay disabled.
- Properties not listed in any step are appended to the last one; an unusable
  `ui:steps` falls back to the ordinary one-page form with a console warning.
- Supports root-level `properties` + `required` + `dependencies`; root-level
  `oneOf`/`allOf` compositions are not split into steps. SDK version
  requirements in `references/advanced.md`.

### Chaining processes (`nextProcessId`)

Each step is its own process. Return the next process's **slug** to advance:

```js
return { nextProcessId: "collect-shipping-address" };
```

The field name is still `nextProcessId` and it accepts a slug or an id — use the
slug, so the same chain works in every workspace.

The SDK then renders the form for `nextProcessId`. Each later step receives all
previous steps' data and results in `fcode.context.parameters` under a `steps`
array. Return a `variables` node alongside `nextProcessId` to pass state forward.

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
prefer `"isSensitive": true` on the property — it renders a password widget
automatically; see `fcode-json-schema`.)

On submit the file is **uploaded to Storage before the process starts**, and the
parameter arrives as an `fcode.storage://…` reference (an **array** if multiple
files allowed). Strip the prefix to download:

```js
const { context: { parameters } } = fcode;
const stream = await fcode.storage.download(
  parameters.inputFile.replace("fcode.storage://", "")
);
```

Uploaded files count toward storage limits — delete them at the end of the
process if only needed transiently.

## Advanced

For styling and the two themes (including the f0 theme to use when embedding in a
React app inside Factorial), markdown message rendering with f0 components
(`markdownComponents`), initial/hidden values, async submission, custom
headers, API-host override, variables replacement, pre-rendering current values
into install/settings forms, reacting to user input from your own page (the
React `onChange` prop, the `fcode-forms-*` DOM events), and modal rendering,
read `references/advanced.md`.
