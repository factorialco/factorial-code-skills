# Factorial Code Forms — advanced

Read this for what you control of a form's look, variables, the server-side
pre-render, and reacting to user input. Enabling, where forms appear,
results, multi-step, and file uploads are in `SKILL.md`; translating form text
is in `fcode-i18n`.

## How forms look inside Factorial

Factorial draws every form with its own design system (f0), so a form matches
the product around it with no styling of yours — there is no stylesheet,
theme or script to bring. What you control lives in the schema:

- **Fields and widgets** — each property's `ui` object, with the
  [react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form/docs/)
  (rjsf **6**) `uiSchema` keys and Factorial Code's own widgets
  (`fcode-json-schema`; tables, pairing tables, OAuth, action buttons in
  `SKILL.md`).
- **Text** — titles, descriptions and `markdown` blocks in markdown (raw HTML
  is dropped), translated with `fcode.i18n` tokens.
- **The form's heading** — its root `title` / `description` (or `ui:title` /
  `ui:description` in the root `ui`); `"ui:label": false` there hides it.
- **Buttons** — `"ui": { "ui:submitButtonOptions": { "submitText": "Save", "variant": "outline" } }`
  and action buttons (`SKILL.md` "Submit and action buttons").
- **Layout** — steps, as a wizard or tabs, in a strip or a sidebar
  (`SKILL.md` "Multi-step forms").
- **The submit overlay message** — `embedFormOptions.loadingOverlayContent` at
  the schema root, shown under the spinner while a submission runs.

How the f0 rendering maps a few fields: a **file** field is a dropzone with
one chip per picked file (name, size, remove), its text overridable with
`ui:options.dropzoneText`; **`range`** renders as a number input rather than a
slider; the `sourceCode` field keeps its (non-f0) Monaco editor; `radio` maps
to f0's card select, because f0 has no radio control. A **select with no
`default` opens empty** (the user must pick); every other `const` opens as its
default.

## Variables

A schema can hold placeholders filled in each time the form is served:

- **Mustache tokens** inside any text, default or list — `{{newPlan}}`, or a
  section such as `{{#benefits}}* {{.}}\n{{/benefits}}`.
- **`$ref` references** that replace a whole node — `{"$ref": "#/variables/name"}`
  in place of a property, an `enum`, a `default`, a title or a `markdown`
  object.

```json
{
  "title": "Upgrade to {{newPlan}}",
  "type": "object",
  "properties": {
    "team": { "title": "Team to upgrade", "type": "string", "enum": { "$ref": "#/variables/teamNames" } }
  },
  "embedFormOptions": {
    "variables": { "newPlan": "Growth", "teamNames": [] }
  }
}
```

Values come from, in increasing order of precedence:

1. **`embedFormOptions.variables`** in the schema itself — fixed values, and
   the fallbacks used when nothing else sets a variable. **Only this key is
   read**: a `variables` key at the schema root is ignored by the form, so its
   values never reach a `$ref` or a token.
2. The `variables` a `preRenderProcess` returns (merged into
   `embedFormOptions.variables` when the schema is served).
3. The `variables` an earlier step returned, in a multi-process flow or a
   `nextProcessId` chain (`SKILL.md` "Multi-step forms"). Only these are
   sent back on submit, in `fcode.context.parameters.metadata.variables`.

Two rules:

- **Use `$ref` for values that must stay intact**, such as URLs. Mustache
  HTML-escapes what it inserts (`/` → `&#x2F;`), so an OAuth `authorizationUrl`
  inserted with `{{…}}` is refused by the connect button.
- **A `$ref` to a missing or empty variable removes the node.** A reference
  whose variable is absent, `null`, `false`, `0` or `""` is dropped from the
  schema, together with any object left empty. Set a variable to `null` to
  remove a property on purpose, and never `$ref` a value that may legitimately
  be `0`, `false` or `""` (a numeric default, a boolean): use a Mustache token
  or a literal, or wrap it in an object (`{ "before": "" }` survives, `""`
  doesn't).

## Server-side pre-render (`preRenderProcess`)

To compute `variables` on the server before the form is shown (a dropdown loaded
from an API, config from team variables), add a `preRenderProcess` at the schema
root set to a process slug/id. When the form is served the API runs that process
**synchronously** and merges the `variables` it returns into the schema, so the
`$ref`s resolve — without a throwaway first step. The process must return
`{ variables: { ... } }`; they land in `embedFormOptions.variables`, on top of
the schema's own fallbacks. The form's context arrives as
`fcode.context.parameters` (never trusted, below). Since it runs before any user input, it can't use
data derived from user-submitted secrets — for that, put the secret in an
earlier step of a multi-process flow and give the pre-render to the next
step's form (`SKILL.md` "Multi-step forms").
It is also where an OAuth connect field's `authorizationUrl` comes from: the
pre-render calls `fcode.oauth.start()` and injects the `authorizationUrl` it
returns. The platform mints the `state` and the PKCE pair, so never build
either by hand (see `SKILL.md` "Connect an external account").
The form is only served once the pre-render finishes, so keep it fast.

### Who is opening the form (`fcode.context.factorial`)

**`fcode.context.parameters` are never trusted** in a pre-render: on a form
opened from Factorial they are the form's context as the browser chose it, and
on a public form they are whatever the request sent. An `access_id` or
`company_id` there proves nothing.

**When Factorial reads the schema for a person** (an action form, a
form-backed UI trigger, Factorial One), the pre-render also gets
`fcode.context.factorial`, the same trusted object the form's submit process
gets when Factorial invokes it: `{ company_id, access_id, entry_point: { type,
location_id? }, product, actor: { service?, type: "user", access_id } }`. The
platform sets it from Factorial's own authenticated request, never from the
parameters, and the actor is always a person (`type: "user"`). It is **absent**
(`undefined` in JavaScript, `None` in Python) when the form is served from its
public link, and when a service reads the schema without naming a
person's access. Treat absence as "unknown viewer" and render the restrictive
version.

Use it to adapt the form to the person, e.g. only let the teams picked in the
App's settings see the edit field. **Leave the field out instead of hiding it
with `ui`**: Factorial One (`entry_point.type` `agent_tool`) gets the schema
without `ui`, `ui:*` and `markdown`, so a `ui:widget: hidden` field is still a
plain, fillable field there. A property whose `#/variables` reference resolves
to `null` is dropped on every surface, One included:

```json
{
  "type": "object",
  "preRenderProcess": "menu-edit-prerender",
  "embedFormOptions": { "variables": { "notice": { "before": "" } } },
  "markdown": { "$ref": "#/variables/notice" },
  "properties": {
    "menu": { "$ref": "#/variables/menuField" }
  }
}
```

Keep such a field out of `required`. The notice sits on the root schema, so the
form still shows it when the field is gone (One never shows `markdown`).

```javascript
// processes/menu-edit-prerender/index.js
async function main() {
  const factorial = fcode.context.factorial; // never fcode.context.parameters
  // Access ids of the teams chosen in settings, resolved and stored by the settings process.
  const allowed = JSON.parse((await fcode.datastore.get("menu.editorAccessIds")) || "[]");
  const canEdit = factorial !== undefined && allowed.includes(String(factorial.access_id));

  return {
    variables: {
      menuField: canEdit ? { type: "string", title: "Today's menu" } : null,
      notice: { before: canEdit ? "" : "Only the menu team can update the menu." },
    },
  };
}
```

Two limits to state in the App, not work around:

- **Leaving a field out is UX only.** Anyone can still submit the form with
  any values, so the **submit process re-checks** the same rule against *its own*
  `fcode.context.factorial` and refuses the rest.
- **The form is still listed for everyone** on the App page in Factorial. A
  pre-render can change what the form shows, not who sees it in the list.

### Pre-filling current values (install & settings forms)

An `install` or `settings` form (the reserved lifecycle slugs) re-opened later should **show what is configured
now**, not an empty form. The schema is static, so the pre-render process is
the place: read the current state from team variables and the datastore, and
return it as `#/variables` nodes the schema's `default`s point at. A worked
example is in `fcode-examples` (`references/custom-app-linear.md`).

```json
{
  "type": "object",
  "preRenderProcess": "acme-settings-prerender",
  "embedFormOptions": {
    "variables": {
      "syncIntervalDefault": 60,
      "apiKeyLabel": "Acme API key",
      "statusMarkdown": { "before": "" }
    }
  },
  "properties": {
    "sync_interval_minutes": {
      "type": "integer",
      "title": "Sync interval (minutes)",
      "minimum": 1,
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
      // Plain config round-trips as the field's default. A stored 0 would drop the
      // default (a `$ref` to 0 removes the node), so the schema's minimum is 1.
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

**Never send a stored secret back to the form** — the schema response lands in
the browser DOM. Report *whether* a credential is set (label, description, or a
`markdown` status line, as above) and let a blank submit mean "keep the current
value": the submit handler writes the variable only when the field is
non-empty, and the field must **not** be `required`, or a user changing another
setting can't submit. The platform backs this up: when it serves a schema it
removes `default`, `examples` and `const` from every `isSensitive` field (and
from everything below one, plus the `variables` such a value referenced), so a
secret returned as a field default never leaves the server. It cannot see a
secret written into a label, a description, a `markdown` block, a field that
isn't `isSensitive` or a `variables` node used elsewhere — those still reach the
browser, so the rule stands. Two more things:

- **Read variables through `fcode.env` / `process.env`, not by listing them.**
  A pre-render sees inherited variables from parent workspaces too
  (`fcode-core-concepts`), so a value can be configured without this workspace
  owning it — which is exactly what should be shown as "configured".
- **A pre-render failure fails the form load**, so a settings form whose
  pre-render throws becomes unopenable. Default missing state instead of throwing
  (`?? 60`, `|| "{}"`), and reserve throwing for genuinely unusable state.

## Reacting to user input

A schema cannot run code in the browser, so a form cannot reformat a value or
compute one field from another while the user types, and there is no page of
yours around it to hook into. Use what the schema and the server offer:

- **Show or hide fields** from another field's value with conditional fields
  (top-level `dependencies`, `fcode-json-schema`).
- **Round-trip to the server** with an action button (such as **Check
  connection**) whose process answers with a message or inline errors, or with
  a multi-step form whose next step is built from what the previous one stored
  (`SKILL.md`).
- **Normalise values in the process** that receives the submission, which has
  to check them anyway.
