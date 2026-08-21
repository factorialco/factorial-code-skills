# Factorial Code Forms — advanced

Read this when styling/theming a form, reacting to user input client-side, or
rendering the form in a modal. Core embedding, result handling, multi-step,
and file uploads are in `SKILL.md`; translating form text is in `fcode-i18n`.

## Initial values, async, headers, API host

Configurable via data attributes, `Fcode.initForm` options, or React props:

- **Process version** — `processVersion` (`data-fcode-form-process-version`): a
  version tag or alias the form loads and submits against — always pin `stable`
  (see `SKILL.md`).
- **Initial / hidden values** — `defaultValues` (`data-fcode-form-default-values`):
  JSON of pre-filled field values.
- **Async execution** — `async: true` (`data-fcode-form-async`): returns `201` +
  execution ID immediately instead of waiting (use for long-running processes).
- **Submission headers** — `headers` (`data-fcode-form-headers`): extra request
  headers.
- **Locale** — `locale` (`data-fcode-form-locale`): the language the form loads
  and submits in, sent as the `Fcode-Locale` header; changing it refetches the
  schema. Model in `fcode-i18n`.
- **API host override** — `hostUrl` (`data-fcode-form-host-url`): point the embed
  at a different backend (default `https://code.factorialhr.com/platform`).

## Styling & appearance

Configured via the embed-side `options` object or `embedFormOptions` in the
schema:

```json
{
  "theme": "light (default)",
  "loadingOverlayDisabled": false,
  "loadingOverlayContent": "Sending information...",
  "loadingContent": "Loading form..."
}
```

- **`theme`** — `"light"` (default) or `"none"`. `light` injects the SDK's
  standalone stylesheet
  (`https://code.factorialhr.com/sdk/styles-theme-light.css`); `"none"` injects
  nothing, which is what you want when the host app brings its own styling (see
  the f0 theme below). Custom theme: extend that CSS file and set
  `embedFormOptions.themeStylesheet` (a URL or inline CSS).
- **`loadingContent`** — the message shown while the form itself is loading (as
  opposed to `loadingOverlayContent`, shown while a submission runs). Worth
  setting when a `preRenderProcess` makes opening the form slow.
- **CSS hooks** — `.fcode-form-container`, `.fcode-form-wrapper`,
  `form.fcode-form`; the result views that replace the form after submit are
  `.fcode-form-success-message` / `.fcode-form-error-message`; a `ui:steps`
  form's navigation uses `.fcode-steps-*` classes; set
  `embedFormOptions.className` for a custom wrapper class.
- **Submit button text** — in the schema:
  `"ui": { "ui:submitButtonOptions": { "submitText": "Click me!" } }`.

Forms render with
[react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form/docs/)
(rjsf **6**), so its full `uiSchema` is available. Titles, descriptions and help
texts support inline markdown (links, emphasis, images); block content — a
field's `markdown.before`/`markdown.after` and the result `message` /
`errorMessage` — additionally supports full GitHub Flavored Markdown, including
tables and code blocks (see `SKILL.md`).

## The f0 theme — for React apps inside Factorial

There are two rendering themes, and which one to use depends on where the form
is embedded:

| Where the form renders | Theme |
|---|---|
| Inside Factorial (the Factorial Code dashboard, the monolith) — anything already running f0 | **`@factorialco/rjsf-f0`** — real f0 components |
| An arbitrary third-party page, via the hosted `forms.js` embed | The SDK's built-in stylesheet theme (`light`) |

**Inside Factorial, use the f0 theme.** The form then renders with actual f0
controls rather than plain HTML styled to imitate them, so it matches the
surrounding product:

```jsx
import FcodeForm from "@factorialco/fcode-react-forms";
import { Theme, markdownComponents } from "@factorialco/rjsf-f0";
import "@factorialco/rjsf-f0/styles.css";

<FcodeForm
  team="<fcode-team-slug>"
  processId="<fcode-process-slug>"
  processVersion="stable"
  rjsfTheme={Theme}
  markdownComponents={markdownComponents}
  // theme: "none" stops the SDK injecting its standalone stylesheet, whose
  // `all: revert` reset exists to survive third-party pages and would fight
  // the host app's own styles.
  options={{ theme: "none", locale: "en" }}
/>;
```

`markdownComponents` (exported since `@factorialco/rjsf-f0` 2.0.0) makes result
messages and a schema's `markdown` blocks render with f0 components too — a GFM
table in a `message` becomes the f0 table card, complete with its Excel/CSV
export (the export lazy-loads `xlsx` through f0 at click time). Omit the prop
and the same markdown renders as plain elements styled by whatever theme
applies.

- The host app must already render f0's `F0Provider` above the form and import
  `@factorialco/f0-react/dist/styles.css`. The theme package ships only the layout
  rules for the parts rjsf composes.
- Peers: `react`, `react-dom`, `@rjsf/core` 6, `@rjsf/utils` 6,
  `@factorialco/f0-react` 4.39+.
- Also exported: `Theme`, `Templates`, `Widgets`, `generateTheme()`,
  `generateForm()`, and a default `Form` (`withTheme(Theme)`) for use as a plain
  rjsf theme without the fcode SDK — the same surface as the official `@rjsf/*`
  theme packages.
- `rjsfTheme` swaps the theme wholesale; explicit `templates`/`widgets`/`fields`
  props still win. Merge order: default theme < `rjsfTheme` < individual props.
- Since 2.1.0 the theme also carries the `ui:steps` navigation (see `SKILL.md`):
  the sidebar layout renders as f0's table of contents, tabs as an f0 button
  strip — nothing extra to wire, `rjsfTheme={Theme}` brings it along.

**Never pull the f0 theme into a hosted-embed page.** f0 cannot be tree-shaken —
importing a single field costs ~3.2 MB gzip, against 262 KB for the whole hosted
bundle. The hosted `forms.js` bundle is size-checked in CI precisely to keep f0
out of it. Only the React component path can opt in.

Three gaps in the f0 mapping worth knowing, since they look inconsistent in a
rendered form: **file fields** keep rjsf's native `<input type="file">`,
**`range`** renders as a number input rather than a slider, and the SDK's
`sourceCode` field keeps its (non-f0) Monaco editor. `radio` maps to f0's
`cardSelect`, because f0 has no radio control.

## Variables replacement

Provide a `variables` node inside `options` to replace
[mustache](https://mustache.github.io/) tokens across the schema (titles,
descriptions, defaults, enums):

```json
{ "title": "Upgrade to {{newPlan}} plan", "description": "{{#benefits}}* {{.}}\n{{/benefits}}" }
```

Use `$ref` to `#/variables/<name>` to replace whole schema nodes (e.g. an
`enum`'s options) per embed. Variables are also sent on submit — read them in
process code via `fcode.context.parameters.metadata.variables`.

### Server-side pre-render (`preRenderProcess`)

To compute `variables` on the server before the form is shown (a dropdown loaded
from an API, config from team variables), add a `preRenderProcess` at the schema
root set to a process slug/id. When the form is served the API runs that process
**synchronously** and merges the `variables` it returns into the schema, so the
`$ref`s resolve — without a throwaway first step. The process must return
`{ variables: { ... } }`. Form query-string params arrive as
`fcode.context.parameters`. Failure/timeout fails the form load. Since it runs
before any user input, it can't use data derived from user-submitted secrets —
that still needs a multi-step form.

Because the form is only served once the pre-render finishes, opening it takes as
long as the process runs — set `loadingContent` (above) to say what's loading.

### Pre-filling current values (install & settings forms)

A `SETTINGS` form re-opened after install, or an `INSTALL` form re-opened to fix a
value, should **show what is configured now** — not an empty form the user has to
fill from memory. Since the schema is static and the embed lives inside Factorial
(so you can't set `defaultValues` on it), the pre-render process is the place to
do this: read the current state from team variables and the datastore, and return
it as `#/variables` nodes the schema's `default`s point at.

Give any `INSTALL` or `SETTINGS` process a `preRenderProcess` when it has values
worth showing back. A worked example is in `fcode-examples`
(`references/custom-app-linear.md`).

```json
{
  "type": "object",
  "preRenderProcess": "acme-settings-prerender",
  "variables": {
    "syncIntervalDefault": 60,
    "apiKeyLabel": "Acme API key",
    "statusMarkdown": { "before": "" }
  },
  "properties": {
    "sync_interval_minutes": {
      "type": "integer",
      "title": "Sync interval (minutes)",
      "default": { "$ref": "#/variables/syncIntervalDefault" }
    },
    "acme_api_key": {
      "type": "string",
      "isSensitive": true,
      "title": { "$ref": "#/variables/apiKeyLabel" },
      "markdown": { "$ref": "#/variables/statusMarkdown" }
    }
  }
}
```

```javascript
// processes/acme-settings-prerender/index.js
async function main() {
  const configured = Boolean(process.env.ACME_API_KEY);
  const stored = JSON.parse((await fcode.datastore.get("acme.settings")) || "{}");

  return {
    variables: {
      // Plain config round-trips as the field's default.
      syncIntervalDefault: stored.syncIntervalMinutes ?? 60,
      // A secret is NEVER echoed — only whether one exists.
      apiKeyLabel: configured
        ? "Acme API key (configured — leave blank to keep the current one)"
        : "Acme API key",
      statusMarkdown: {
        before: configured
          ? "Connected."
          : "**Not connected yet.** Enter an API key to finish setup.",
      },
    },
  };
}
```

**Never send a stored secret back to the form.** The schema response travels over
HTTP and lands in the browser DOM. Report *whether* a credential is set — in the
label, the description, or a `markdown` status line — and let a blank submit mean
"keep the current value":

```javascript
// In the settings process that handles the submit:
const { acme_api_key } = fcode.context.parameters;
if (acme_api_key) {
  await fcode.variables.set("ACME_API_KEY", acme_api_key); // rotate
}
// blank → keep whatever is stored; don't overwrite with ""
```

That also means such a field must **not** be `required` in the schema, or a user
who only wants to change the sync interval can't submit. Two more things:

- **Read variables through `fcode.env` / `process.env`, not by listing them.**
  A pre-render sees inherited variables from parent workspaces too
  (`fcode-core-concepts`), so a value can be configured without this workspace
  owning it — which is exactly what should be shown as "configured".
- **A pre-render failure fails the form load**, so a settings form whose
  pre-render throws becomes unopenable. Default missing state instead of throwing
  (`?? 60`, `|| "{}"`), and reserve throwing for genuinely unusable state.

## Internationalization

Visible text is translated by writing `fcode.i18n("key")` tokens in the schema,
substituted server-side from the workspace's locales before the schema is
served — full model and migration guide in `fcode-i18n`.

## Reacting to user input

**A schema cannot carry executable code.** It is served to every visitor of the
form, so `embedFormOptions.onChange` and `embedFormOptions.fields.<field>.transformFn`
were removed in `@factorialco/fcode-react-forms` 2.0.0. Reformatting a value or
deriving one field from another belongs in the embedding page:

- **React embed** — the `onChange` prop on `FcodeForm`.
- **Hosted script** — the events the SDK dispatches on `document`:
  `fcode-forms-sdk-init`, `fcode-forms-init-form`,
  `fcode-forms-on-submit-success`, `fcode-forms-on-next-step` and
  `fcode-forms-on-submit-error`.

```html
<script>
  document.addEventListener("fcode-forms-on-submit-success", (event) => {
    const { formId, formSubmittedData, processExecutionResult } = event.detail;
    analytics.track("User Registered", formSubmittedData);
  });
</script>
```

For submission results specifically, the `onSuccess` / `onNextStep` / `onError`
callbacks (see `SKILL.md`) carry the same information.

## Messages are markdown, not HTML

`markdown.before`/`markdown.after` in a schema, the result `message`, and error
messages (`errorMessage`) render as GitHub Flavored Markdown — tables, headings,
links, images, code blocks, task lists. Raw HTML tags are dropped, never
rendered, so `<script>` tags and inline `on*` handlers can never execute, and
`javascript:` URLs are neutralized.

CSS injection via `stylesheet` / `themeStylesheet` is unaffected.

## Modal window

Render the form in a modal triggered by an element: add all the form config plus
the `data-fcode-form-modal` attribute to that element.

```html
<button
  data-fcode-form-modal
  data-fcode-form-team="<fcode-team-slug>"
  data-fcode-form-process="<fcode-process-slug>"
  data-fcode-form-process-version="stable"
>
  Open the form
</button>
```
