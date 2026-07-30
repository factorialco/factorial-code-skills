# Factorial Code Forms — advanced

Read this when styling/theming a form, doing internationalization, reacting to
user input client-side, or rendering the form in a modal. Core embedding,
result handling, multi-step, and file uploads are in `SKILL.md`.

## Initial values, async, headers, API host

Configurable via data attributes, `Fcode.initForm` options, or React props:

- **Initial / hidden values** — `defaultValues` (`data-fcode-form-default-values`):
  JSON of pre-filled field values.
- **Async execution** — `async: true` (`data-fcode-form-async`): returns `201` +
  execution ID immediately instead of waiting (use for long-running processes).
- **Submission headers** — `headers` (`data-fcode-form-headers`): extra request
  headers.
- **API host override** — `hostUrl` (`data-fcode-form-host-url`): point the embed
  at a different backend (default `https://code.factorial.dev/platform`).

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

- **Themes** — one built-in theme, `light`
  (`https://code.factorial.dev/sdk/styles-theme-light.css`). Custom theme:
  extend that CSS file and set `embedFormOptions.themeStylesheet` (a URL or
  inline CSS).
- **`loadingContent`** — the message shown while the form itself is loading (as
  opposed to `loadingOverlayContent`, shown while a submission runs). Worth
  setting when a `preRenderProcess` makes opening the form slow.
- **CSS hooks** — `.fcode-form-container`, `.fcode-form-wrapper`,
  `form.fcode-form`; set `embedFormOptions.className` for a custom wrapper class.
- **Submit button text** — in the schema:
  `"ui": { "ui:submitButtonOptions": { "submitText": "Click me!" } }`.

Forms render with
[react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form/docs/),
so its full `uiSchema` is available (plus markdown in titles/descriptions/help).

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

## Internationalization

Use mustache tokens for visible text and supply `i18nVariables` per locale inside
`embedFormOptions`, then set `locale` (and optional `fallbackLocale`) in the
embed `options`:

```json
"embedFormOptions": {
  "i18nVariables": {
    "en": { "title": "Signup form" },
    "es": { "title": "Formulario de registro" }
  }
}
```

The selected locale is sent on submit — available in process code as
`fcode.context.parameters.metadata.locale`.

## Reacting to user input

**A schema cannot carry executable code.** It is served to every visitor of the
form, so `embedFormOptions.onChange` and `embedFormOptions.fields.<field>.transformFn`
were removed in `@factorialco/fcode-react-forms` 2.0.0, and a `jsCallback`
returned by a process execution is ignored. Reformatting a value or deriving one
field from another belongs in the embedding page:

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

## Authored HTML is sanitized

`rawHtml.before`/`rawHtml.after` in a schema, the result `message`, and error
messages all pass through DOMPurify before rendering. Markup renders, but
`<script>` tags, inline `on*` handlers and `javascript:` URLs do not execute.
`iframe`, `object`, `embed`, `base` and `form` elements are forbidden, as are the
`srcdoc` and `formaction` attributes, and `target="_blank"` links are forced to
`rel="noreferrer"`. Dropped content is reported as a console warning.

CSS injection via `stylesheet` / `themeStylesheet` is unaffected.

## Modal window

Render the form in a modal triggered by an element: add all the form config plus
the `data-fcode-form-modal` attribute to that element.

```html
<button
  data-fcode-form-modal
  data-fcode-form-team="<fcode-team-slug>"
  data-fcode-form-process="<fcode-process-slug>"
>
  Open the form
</button>
```
