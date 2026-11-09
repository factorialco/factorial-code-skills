---
name: fcode-json-schema
description: Author Factorial Code process input-parameter schemas (parametersSchema.json) — every supported input type and validation option, with a complete annotated sample. Use when creating or editing a parametersSchema.json, or defining a Factorial Code (fcode) process's input parameters or form fields.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — input parameter schemas

A process's input parameters are defined by a JSON Schema in
`parametersSchema.json`. The same schema is what `fcode-forms` renders as a web
form, so designing the schema *is* designing the form. Values arrive in code as
`fcode.context.parameters` (see `fcode-javascript` / `fcode-python`).

Schemas render with [react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form/),
so its `ui:` options apply.

## How to author one

1. Top level is `"type": "object"` with a `properties` map (one entry per field)
   and an optional `required` array.
2. Pick a `type` per field (`string`, `integer`, `number`, `boolean`, `array`,
   `object`) and add validation (`minimum`/`maximum`, `format`, `enum`/`oneOf`,
   `uniqueItems`, nested `properties`).
3. Control rendering with a per-field `ui` object (e.g.
   `"ui": { "ui:widget": "textarea" }`).
4. **Open `assets/parametersSchema.sample.json`** for a complete, copy-pasteable
   schema exercising every supported type — adapt fields from it rather than
   guessing the shape.

## Field types & widgets (in the sample)

- **Text** — `"type": "string"`; `format: "email"`; `ui:widget` of `textarea`,
  `color`, `hidden`, or `file`.
- **Secret** — `"isSensitive": true` masks the input and marks it as a secret
  for the platform (e.g. passwords/tokens); the platform never serves its
  `default`/`examples`/`const` (see `fcode-forms`). `"ui:widget": "password"`
  alone only hides the value on screen — without `isSensitive` the field is not
  protected, and the CLI warns on push about a password widget or a
  secret-looking name (token, password, API key, client secret…) without it.
- **Numbers** — `integer` / `number` with `minimum` / `maximum`.
- **Boolean** — `"type": "boolean"`.
- **Choices** — `enum` (select), `oneOf` of `{ const, title }` (labeled radio
  with `ui:widget: "radio"`), or an array with `ui:widget: "checkboxes"` +
  `uniqueItems`.
- **Structured** — nested `object` with its own `properties`/`required`; arrays
  of strings or of objects (`type: "array"` + `items`); raw JSON via
  `"ui": { "ui:field": "json" }`.
- **Conditional fields** — use top-level `dependencies` to show/hide fields based
  on another field's value (see `anotherBooleanField` in the sample).
- **Content around a field** — a per-field `markdown` object with `before` /
  `after` strings renders block markdown (full GFM, incl. tables) above/below
  the field. Detail and a worked example in `fcode-forms`
  (`references/advanced.md`).
- **Long forms** — split roughly ten or more fields into steps with a root-level
  `ui:steps` node; see `fcode-forms` §Multi-step forms. The same node can
  declare `process` steps that span several processes' forms (a multi-process
  flow, owned by `fcode-forms`).
- **Connect an external account** — a `string` or `boolean` with
  `"ui": { "ui:widget": "oauth", "ui:options": { "authorizationUrl": … } }`
  renders a button that completes an OAuth flow in a popup (`"ui:field":
  "oauth"` for an `object`; see `oneOauthConnectField` in the sample). Contract
  and callback in `fcode-forms` §Connect an external account.

## Gotchas

- **A file field** (`"ui:widget": "file"`) is auto-uploaded to Storage before
  the process runs; the parameter arrives as an `fcode.storage://…` reference,
  not the file contents. Strip the prefix before `fcode.storage.download(...)`.
- **`isSensitive: true`** only affects display/masking — still read the value
  from a secret variable, never hardcode it.
- **An oauth field's `authorizationUrl` must be injected with
  `{ "$ref": "#/variables/x" }`** from a `preRenderProcess` variable — never
  `{{x}}`, which Mustache HTML-escapes into a URL the widget refuses (the
  button renders disabled with a console warning).
- The schema is the single source of the form's fields — to change fields, edit
  the schema, not the form embed code.
- **The schema is data, not code** — no executable JavaScript, and all form
  text is markdown (raw HTML is never rendered). Client-side behaviour belongs
  in the embedding page — see `fcode-forms`.
- **Size a choice field for the biggest customer.** Every option of an
  `enum`, a `oneOf` of `{ const, title }` or the `fcode-forms` module's
  `selectField` / `multiSelectField` ships inside the schema, so a list fed
  from a directory that grows with the company (every employee, every
  manager, all projects or cost centers) turns into thousands of options for
  a large account. Nobody can pick from that, and hosts on a forms SDK older
  than `@factorialco/fcode-react-forms` 3.7.1 fail validation with "Maximum
  call stack size exceeded" from about 1,000 options: the submit button
  silently does nothing. The forms SDK has no searchable or remote-loading
  picker, so ask for something bounded instead: a team, legal entity, role or
  location the process resolves to people; a list narrowed by the API's
  filters to the people who can plausibly be chosen (keep it to a couple of
  hundred at most); or emails / ids as text that the process checks and
  rejects with a clear message.
- **Visible text can be translated**: schema strings (titles, descriptions,
  `ui:placeholder`) accept `fcode.i18n("key")` tokens, substituted server-side
  before the form is served — see `fcode-i18n`.
