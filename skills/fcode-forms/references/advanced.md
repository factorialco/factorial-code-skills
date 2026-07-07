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
  "theme": "dark (default) | light",
  "loadingOverlayDisabled": false,
  "loadingOverlayContent": "Sending information..."
}
```

- **Themes** — built-in `dark` and `light`. Custom theme: extend a theme CSS file
  and set `embedFormOptions.themeStylesheet` (a URL or inline CSS).
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

## Behaviour functions

Inside `embedFormOptions`, provide JavaScript (as a string) to react to input:

- **`onChange`** — receives `formData` and `setFormData`; mutate and call
  `setFormData(formData)`.

  ```json
  "embedFormOptions": { "onChange": "formData['phonePrefix'] = {'ES':'+34','US':'+1'}[formData['countryCode']]; setFormData(formData);" }
  ```

- **Field transformer** — `embedFormOptions.fields.<field>.transformFn` receives
  `value`, returns the new value.

  ```json
  "embedFormOptions": { "fields": { "phone": { "transformFn": "return value && value.trim();" } } }
  ```

## Modal window

Render the form in a modal triggered by an element: add all the form config plus
the `data-fcode-form-modal` attribute to that element.

```html
<button
  data-fcode-form-modal
  data-fcode-form-team="<fcode-team-id>"
  data-fcode-form-process="<fcode-process-id>"
>
  Open the form
</button>
```
