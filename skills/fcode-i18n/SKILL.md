---
name: fcode-i18n
description: Workspace locales and translations for Factorial Code — i18n/<locale>.yaml files, the fcode.i18n helper in code and in form schemas, execution-locale selection, per-call locale overrides, primary-locale fallback, locale versioning, and the i18n:* CLI commands. Use when adding a locale, internationalizing process code or form text, rendering several languages in one execution, or testing and syncing translations.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — i18n

A workspace keeps one **locale** per language it speaks: a YAML file of
translation keys under `i18n/`, synced by the CLI like any other resource.
`fcode.i18n("key")` resolves a key against the locale the current execution is
using — in process code, in module code, and (substituted server-side) in form
schemas. Platform model in `fcode-core-concepts`; CLI flow in `fcode-cli`; form
embedding in `fcode-forms`.

## Gotchas

- **Never alias `fcode.i18n`** — translations are only shipped to an execution
  when `fcode.i18n(` is statically detected in the source, so
  `const t = fcode.i18n; t("k")` throws **"i18n is disabled"** at runtime.
  Always call it literally, with the key as a hardcoded string (same class of
  rule as `fcode.import` module names).
- **The helper never fails.** A key with no translation anywhere resolves to
  **the key itself** — a raw `greetings.hello` in output means a missing
  translation, never a broken run. A placeholder you pass no argument for is
  left exactly as written; a non-object `args` (including arrays) is ignored.
- **Locale identifiers are case-sensitive** (`pt-BR` ≠ `pt-br`) and this is
  permanent platform-wide. Valid: up to 20 letters, numbers, `-` or `_`,
  starting with a letter or number (`^[A-Za-z0-9][A-Za-z0-9_-]{0,19}$`) — no
  dots. On a case-folding filesystem (macOS/Windows defaults) the CLI refuses
  to pull two locales differing only in case, since their files would collapse
  into one.
- **Never edit `i18n/<locale>.inherited.yaml`** — read-only, gitignored,
  regenerated on pull. To override an inherited key, write it into your own
  `i18n/<locale>.yaml`: overrides layer **key by key**, never file by file, so
  keys you don't mention keep resolving to the parent's text.
- **`locale` is a reserved name** on form and webhook endpoints, like
  `version_tag` and `async`: it selects the language and is stripped before
  the parameters are built. A form field or webhook body field named `locale`
  never reaches the process — use another name for business data.
- **Form-token arguments must be a flat object of scalars** —
  `{ "max": "500" }` works, `{ "max": { "chars": "500" } }` does not; a token
  with nested braces is left untouched in the served schema.
- **A mistyped `version` tag resolves against the current files silently** —
  it never blanks output. When released text looks un-frozen, check the pinned
  tag before anything else. Same shape for a per-call `{ locale }` naming a
  locale that doesn't exist: the lookup behaves as if no locale was named.
- **A pinned `version` must be an inline literal string** — the snapshot to
  ship is read from the source, so `{ version: chosenTag }` resolves against
  the current files. The `locale` option has no such rule: its value may be any
  runtime expression, and options built elsewhere still work.
- **A YAML key written without a value counts as untranslated** — it falls
  through to the fallback locale rather than resolving to an empty string.

## Locale files

One YAML mapping of keys to text per locale, at `i18n/<locale>.yaml`. Nesting
is a convenience for whoever writes the file, not a data model: nested keys are
addressed with dots, so these two files are the same locale —

```yaml
# i18n/en.yaml
greetings:
  hello: "Hi %{name}"
  farewell: "See you"
```

```yaml
# identical to the file above
"greetings.hello": "Hi %{name}"
"greetings.farewell": "See you"
```

— which is what lets a child workspace override a single key without repeating
the parent's structure. `%{name}` placeholders are filled from the arguments
passed to the helper. A locale file is capped at 256 KB (the whole merged set
travels with each execution).

What the workspace **inherits** sits alongside what it owns, in
`i18n/<locale>.inherited.yaml` — read-only, gitignored (the CLI adds the
entry). When several parent workspaces define the same locale, the inherited
file holds their merge in the platform's resolution order, rewritten as a flat
mapping of dotted keys under a generated header; with a single parent the file
is kept verbatim, comments included. A local run layers your own file over it
exactly as the cloud does.

## The `fcode.i18n` helper

`fcode.i18n(key, args, options)` — same name and semantics in JavaScript and
Python, available in processes **and** modules (a process reaching it only
through a module still gets its translations):

```javascript
const greeting = fcode.i18n("greetings.hello", { name: "Ada" }); // "Hi Ada" in `en`
fcode.i18n("greetings.farewell");                    // no placeholders → no args
fcode.i18n("legal.terms", null, { version: "v1.0.0" }); // pinned to a published version
fcode.i18n("greetings.hello", { name: "Ada" }, { locale: "es" }); // another locale, this lookup only
const recipient = { name: "Ada", locale: "pt-BR" };               // ...and the value may be dynamic,
fcode.i18n("greetings.hello", { name: recipient.name }, { locale: recipient.locale }); // per recipient
fcode.i18n("legal.terms", null, { version: "v1.0.0", locale: "es" }); // both combine
const locale = fcode.i18n.locale;                    // the execution's locale
```

```python
greeting = fcode.i18n("greetings.hello", {"name": "Ada"})
fcode.i18n("greetings.farewell")
fcode.i18n("legal.terms", None, {"version": "v1.0.0"})
fcode.i18n("greetings.hello", {"name": "Ada"}, {"locale": "es"})
recipient = {"name": "Ada", "locale": "pt-BR"}
fcode.i18n("greetings.hello", {"name": recipient["name"]}, {"locale": recipient["locale"]})
fcode.i18n("legal.terms", None, {"version": "v1.0.0", "locale": "es"})
locale = fcode.i18n.locale
```

- `version` — a locale version tag or alias, as an **inline literal string**
  (see Gotchas); versioning below.
- `locale` — reads **that lookup** in another locale, and the value may be
  dynamic (an employee's language), so one execution can speak several
  languages. A key the named locale hasn't translated falls back to the
  execution's locale, then the primary; a locale that doesn't exist behaves as
  if none was named — never worse than without the option. `fcode.i18n.locale`
  keeps reporting the execution's locale. Form-schema tokens do **not** take
  this option — a render is already in the language the request chose.
- Interpolation is a **single pass over own properties**: a substituted value
  containing `%{...}` is never rescanned (one argument can't reach another),
  and `%{constructor}` resolves nothing. Missing keys and arguments never
  throw (see Gotchas).

## Translating form schemas

Form schemas are rendered by the browser, so there is no runtime to resolve
keys in. Write the same call **as a string** in `parametersSchema.json` — in
titles, descriptions, `ui:placeholder`, `embedFormOptions.loadingOverlayContent`,
any visible text — and the platform substitutes it **before serving the
schema**. The browser receives a schema already written in one language;
translations never reach the client. Substitution runs after `preRenderProcess`,
so text a pre-render injects is translated too.

```json
{
  "type": "object",
  "properties": {
    "reason": {
      "type": "string",
      "title": "fcode.i18n(\"form.reason.label\")",
      "description": "fcode.i18n(\"form.reason.help\", { max: \"500\" })"
    }
  }
}
```

Arguments follow relaxed JavaScript syntax (single quotes, unquoted field
names, trailing commas all accepted) but must stay a **flat object of
scalars** — a nested value leaves the whole token unsubstituted (see Gotchas).

The reader's locale comes from the embed: `locale` in the embed options or the
`data-fcode-form-locale` attribute, sent to the platform as the `Fcode-Locale`
header. Changing it refetches the schema, and the submit carries the same
header, so the execution runs in the language the form was rendered in.
Embedding mechanics in `fcode-forms`. (`fallbackLocale` in the embed options
plays no part here — it only selects the language of rjsf's built-in
validation messages when `locale` isn't one it ships.)

## How the execution locale is chosen

| Trigger | How to choose |
|---|---|
| Form | `?locale=` query parameter or `Fcode-Locale` header (parameter wins) |
| Webhook | `?locale=` query parameter or `Fcode-Locale` header (parameter wins) |
| Run now | Locale selector in the run dialog |
| Schedule | Locale selector when creating or editing the schedule |
| Rerun | Reuses the original execution's stored locale |
| Per call | `locale` in the helper's options — that lookup only, value may be dynamic |

A malformed locale on the public endpoints is a `400`; an unknown-but-valid
one merely falls back. The chosen locale is stored on the execution, which is
why a rerun reproduces the original run's language even if the workspace's
default has moved since.

When nothing names a locale, the workspace's **primary locale** is used —
`primaryLocale` in `settings.json` (set it under Settings → Details, or edit the
file and `fcode settings:push`; field reference in `fcode-cli`) — and when none is
chosen, the first locale alphabetically. The primary locale is **also the
key-level fallback**: a key the chosen locale hasn't translated resolves from
the primary, and only a key missing from both resolves to its own name. So a
partially translated locale still resolves every key — but keep the primary
complete.

## CLI: syncing and testing locales

Locales are a CLI resource like any other:

```sh
fcode i18n:pull            # fetch every locale, inherited ones included
fcode i18n:status          # what changed locally vs the cloud
fcode i18n:add pt-BR       # track a new local file
fcode i18n:push            # create or update in the cloud
fcode i18n:remove pt-BR    # stop tracking it locally
fcode i18n:reset           # discard local changes
```

- Aggregate `fcode pull` / `push` / `status` include locales, so the usual
  whole-workspace commands already cover them.
- **There is no extract command** — moving hardcoded strings into locale files
  is the agent's job (next section); the CLI only syncs the files.
- **Pushing an identifier a parent workspace owns creates an override here**,
  layered key by key — it never edits the parent's file.
- `primaryLocale` lives in `settings.json` and syncs with `fcode settings:push`.

Local runs resolve `fcode.i18n` against the same `i18n/` files, layered
exactly as the cloud does, so `fcode run my-process` behaves like production.
Pass `--locale` to run in a specific one:

```sh
fcode run my-process --locale pt-BR
```

`--locale` is not format-validated locally: a typo silently matches no file
and every key resolves to itself. If a local run shows raw keys, check the
flag's spelling (and case) first.

## Internationalizing existing code

There is no automated extraction — internationalizing a workspace is a code
transformation you perform, with the CLI as the sync vehicle:

1. **Agree scope with the user**: which locales, and which is primary. If
   unset, write `primaryLocale` in `settings.json` and `fcode settings:push`.
2. **Inventory the user-facing strings.** Translate: form-schema titles,
   descriptions, placeholders and `loadingOverlayContent`; result `message`
   strings a form displays (these are markdown — keep any formatting like
   `**bold**`, links or table syntax intact in every locale); email subjects
   and bodies; webhook response bodies end users see. Do **not** translate:
   log messages, developer-facing errors, datastore keys, variable names,
   slugs and identifiers.
3. **Name the keys** in dotted namespaces: `<process-slug>.<area>.<name>`
   (`order-sync.form.title`, `order-sync.email.subject`), with strings shared
   across processes under `common.*`. Extract dynamic parts as
   `%{placeholders}` — never concatenate translated fragments.
4. **Replace each string**: in code with a literal
   `fcode.i18n("key", { args })` call (never aliased, key hardcoded); in
   schemas with the token string (flat scalar args only).
5. **Populate `i18n/<locale>.yaml` for every locale** (`fcode i18n:add` for
   new ones). The primary locale must cover every key — it is the fallback all
   the others lean on.
6. **Test per locale**: `fcode run <slug> --locale <loc>`. A raw dotted key in
   the output is a missing translation; a literal `%{name}` is a missing
   argument.
7. **Push**: `fcode i18n:push` (or aggregate `fcode push`), plus
   `fcode settings:push` if `primaryLocale` changed.

## Versioning locales

Locales are versioned like processes and modules (model in
`fcode-core-concepts`): publishing snapshots the YAML under an immutable tag,
and aliases are movable pointers — but **per locale**: `production` on `en`
and `production` on `es` are two different aliases.

A pinned call (`{ version: "v1.0.0" }`) resolves **per file** in the
inheritance chain: each locale file answers with its snapshot at that tag, or
with its current content when it has no snapshot at that tag, layered key by
key as usual. So a locale created after the version was cut still contributes
its keys, a parent that never published the tag still contributes its text,
and a mistyped tag resolves everything against the current files rather than
blanking output. `{ version: "v1.0.0", locale: "es" }` combine: the snapshot is
read in the requested locale, with the same fallbacks. Deleting a version sends
the calls pinned to it back to the current files; deleting a locale deletes its
versions with it.

One layer resolves differently: a **parent workspace pinned to one of its
versions** answers with its snapshot at the pin, and falls back to that snapshot
rather than to its current file when the call names a tag it never published.
Its live translations never reach the child — see `fcode-core-concepts`.

**A workspace version freezes translations with the release.** Creating one
(`fcode settings:versions:create`, see `fcode-cli`) publishes a version of every
owned locale — locales first, so the pins below have a target — and rewrites
the **published snapshots** so bare `fcode.i18n` calls pin the tag, in process
code, module code, and form schemas:

```javascript
// Working copy (never modified)
fcode.i18n("greetings.hello", { name: "Ada" });
fcode.i18n("welcome", { name: "Ada" }, { locale: "es" });

// Published v1.0.0 snapshot
fcode.i18n("greetings.hello", { name: "Ada" }, { version: "v1.0.0" });
fcode.i18n("welcome", { name: "Ada" }, { version: "v1.0.0", locale: "es" });
```

A call whose options already name a `version` — even a dynamic one — is
considered intentional and left untouched; options naming none (only a
`locale`, an empty object, an explicit `null`/`None`) get the tag spliced in,
so a localized call freezes with the release while its locale stays as
written — a dynamic value (`{ locale: employee.locale }`) is preserved
verbatim too.
(Note the asymmetry with module imports, which are pinned in the string form —
`fcode.import("m", "v1.0.0")` — for compatibility with older executors; don't
"fix" one to look like the other.) Fixing a released typo means publishing
again — snapshots are immutable.

## REST API, SDKs & MCP tools

Locales are addressed by identifier, and `PUT` upserts, so a sync never needs
to know whether the workspace already had the locale:

```
GET    /{team}/rest/locales
GET    /{team}/rest/locales/{locale}
PUT    /{team}/rest/locales/{locale}     body: { "content": "<yaml>" }
DELETE /{team}/rest/locales/{locale}
```

Both SDKs expose the same surface as `FcodeI18n`:

```javascript
import { FcodeI18n } from "@factorialco/fcode-sdk";
const i18n = new FcodeI18n();
await i18n.list();                                        // inherited included
await i18n.set("pt-BR", 'greetings:\n  hello: "Olá %{name}"\n');
await i18n.delete("pt-BR");
```

```python
from fcode_sdk import FcodeI18n
i18n = FcodeI18n()
i18n.list()
i18n.set("pt-BR", 'greetings:\n  hello: "Olá %{name}"\n')
i18n.delete("pt-BR")
```

`set()` on an identifier a parent workspace owns creates an override here —
the same key-by-key layering the CLI push does.

The MCP server exposes `get_locales`, `get_locale`, `save_locale` and
`delete_locale`. **`save_locale` replaces the locale's content entirely** — to
add keys, `get_locale` first and write back the merged YAML.
