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
- **Pair records with another system** — an `object` with
  `"ui": { "ui:field": "pairing" }` whose properties are one row each (a
  `string`; the other system's records as choices, once for the section in
  `x-fcode-choices` or per row as `oneOf` / `enum`, or no choices at all for a
  row the admin types, when those records can't be listed) renders as one reviewable
  table instead of a dropdown per record. Per row,
  `"x-fcode-suggestion": { "value", "reason" }` proposes a match and
  `"x-fcode-in-use": true` narrows the default list; `default` is the stored
  pair. Use it for any mapping of more than a handful of Factorial records
  (accounts, tax codes, cost centers, wage types, absence codes, positions).
  Contract and semantics in `fcode-forms` §Pairing tables; see `accountPairing`
  in the sample.
- **Let One fill the form** — a `"type": "object"` property with
  `"ui": { "ui:field": "launch-one", "ui:options": { "label", "prompt", … } }`
  renders a "Set up with One" button inside Factorial and submits nothing; see
  `fcode-forms` §Set up with One.
- **Employees** — Factorial's employee picker (search, filters, avatars). An
  `array` of `string` picks several people and a `string` picks one; mark it with
  `"x-fcode-entity": "employee"` and draw it with `"ui:widget": "employee"`. The
  value is Factorial **employee IDs** (not access IDs), the same IDs the public
  API and Factorial One use. Optional `ui:options`: `onlyActive` (default `true`;
  `false` lists inactive employees too) and `filters`, the filter names to offer
  (e.g. `["team", "legalEntity"]`). See `employees` in the sample.

  ```json
  "employees": {
    "type": "array",
    "title": "Employees",
    "items": { "type": "string" },
    "x-fcode-entity": "employee",
    "ui": { "ui:widget": "employee", "ui:options": { "filters": ["team", "legalEntity"] } }
  }
  ```

## Gotchas

- **A file field** (`"ui:widget": "file"`) is auto-uploaded to Storage before
  the process runs; the parameter arrives as an `fcode.storage://…` reference,
  not the file contents. Strip the prefix before `fcode.storage.download(...)`.
- **The employee picker only exists inside Factorial.** Anywhere else (an
  embedded form, the hosted `forms.js`) the field falls back to a plain select
  when it lists its choices with `enum` / `oneOf`, and to a short notice
  otherwise (`ui:options.unavailableMessage` replaces the text). Expose a form
  with an employee field through Factorial (`fcode-factorial-actions`).
- **Keep both keys on an employee field.** `x-fcode-entity` says what the field
  means and is what Factorial One reads; `ui:widget` only draws it, and One never
  sees `ui` keys. Without `x-fcode-entity`, One does not know to pass employee IDs.
- **The picker lists the employees the person filling in the form may see**, not
  what the App may see. Check every employee ID the process receives before acting
  on it.
- **There is no "select all".** The user picks each person; the process never
  receives "everyone matching a filter".
- **A pairing row's suggestion is not its `default`.** Put a proposal in
  `x-fcode-suggestion`, never in `default`: a `default` is a stored pair, shown
  as confirmed and submitted as it is, while a suggestion is only submitted
  once the user accepts it. A `"default": ""` is an explicit "not mapped"
  when the choices include a `""` option (without one the SDK drops it), and
  never a blank. Keep both on the row (`x-fcode-*` is what Factorial One
  reads; `ui` only draws the table).
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
  silently does nothing. Small organisational lists (teams, legal entities,
  locations, roles) stay in the tens even for large accounts and are fine to
  list whole. The forms SDK has no searchable or remote-loading picker, so for
  people, projects or cost centers ask for something bounded instead: a team,
  legal entity, role or location the process resolves to people; a list
  narrowed by the API's filters to the people who can plausibly be chosen
  (keep it to a couple of hundred at most); or emails / ids as text. Free text
  drops the limit a select gave, so the process must check both that the
  person exists and that the caller may choose them (for example as an
  approver), and reject anything else with a clear message — parameters are
  untrusted (`fcode-forms` §What a form may trust).
- **Visible text can be translated**: schema strings (titles, descriptions,
  `ui:placeholder`) accept `fcode.i18n("key")` tokens, substituted server-side
  before the form is served — see `fcode-i18n`.
