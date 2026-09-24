---
name: fcode-factorial-actions
description: Expose a Factorial Code process to Factorial as a Factorial Action — the `factorial` block in a process's metadata.json (master switch, awaitResult, requiredPolicies, and the uiTrigger / form / agentTool / backend entry points), the Factorial One tool contract (description, effect, whenToUse, whenNotToUse, preconditions, doesNotDo, degradation, simulatesFor), the reserved lifecycle slugs, the parameters schema as the contract with Factorial, and how the legacy `form` / `uiTrigger` blocks are kept mirrored. Use when configuring how Factorial reaches a process, writing an agent tool contract, or filling the "Trigger from Factorial" section of a process.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Factorial Actions

A **Factorial Action** is a process that the customers who installed an App
reach from **inside Factorial**, without knowing Factorial Code exists. One
process is one action, and the same action can be exposed through several
**entry points** at once — a button in the Factorial UI, a form, a Factorial One
tool, a backend job. Whatever the entry point, Factorial sends the process its
parameters and, when asked to, waits for the result; the process does not know
which entry point started it, so one implementation serves all of them.

Everything an action needs lives in **one settings block, `factorial`**: the
**Trigger from Factorial** section of the process page in the console, or the
`factorial` key of `processes/<slug>/metadata.json` (round-tripped by
`fcode pull` / `fcode push`, field reference in `fcode-cli`).

## What is live today (phase 1) and what is not

Phase 1 ships the **settings block only**. Read the rest of this skill with
that in mind.

| Available now | Coming in later phases |
|---|---|
| The `factorial` block in the console, in `metadata.json`, and as `settings.factorial` in the API | Factorial **invoking** actions through the new trusted path (enforcing `requiredPolicies`, running `backend` jobs, authenticating forms opened from Factorial) |
| The legacy `form` and `uiTrigger` blocks kept **mirrored** with `factorial`, so the button and the form keep working through them | **Factorial One tools**: the assistant reading the `agentTool` contract and calling the process |
| `uiTrigger.label` localized with `fcode.i18n("key")` tokens on the `?locale=` listing | The **dashboard actions API** Factorial will use to list and run a workspace's actions |
| `lifecycleRole` derived from the reserved slugs and tagged in the console | Enforcing `form.public`; retiring what is left of `form.appRole` |

So today: `uiTrigger` and `form` do something (through the mirror), while
`agentTool`, `backend`, `requiredPolicies` and `form.public` are **stored,
not enforced**. Fill them in now so the action is ready when each path lands.

## Gotchas

- **`enabled` is a master switch.** While it is `false` nothing below is
  exposed, whatever each entry point's own `enabled` says. Turning it back on
  restores the entry points exactly as they were.
- **`awaitResult` defaults to `true` (synchronous).** The legacy `uiTrigger`
  defaulted to fire-and-forget. The mirror copies `factorial.awaitResult` into
  `uiTrigger.awaitResult`, so a button moved to the `factorial` block **becomes
  synchronous** unless you write `"awaitResult": false`. Keep synchronous
  actions under the ~50 s budget (`fcode-ui-triggers`).
- **`requiredPolicies` are free text and unchecked.** A key Factorial does not
  recognise is never granted, so the whole AND-group containing it fails — it
  can never *open* an action, only close one. Take keys from Factorial's policy
  catalogue; prefer several small alternatives over one long group.
- **Only `uiTrigger.label` is translatable.** It is the one text users see, so
  it takes `fcode.i18n("key")` tokens (`fcode-i18n`). The `agentTool` texts are
  for the model — plain text, one language, never shown to users. The process's
  own `name` / `description` serve humans and the marketplace card; the block
  has no title of its own.
- **An unset `effect` is treated as `DESTRUCTIVE`.** Every tool fails closed
  (approval + "cannot be undone" warning) until you say `READ` or `WRITE`.
- **The four lifecycle slugs are reserved.** `install`, `settings`,
  `uninstall`, `sync` mean something to Factorial regardless of the block. Do
  not give an ordinary action one of them.
- **Omit the defaults.** `fcode pull` leaves out `awaitResult` when `true`,
  `requiredPolicies` when empty, `form.public` and `form.appTool` when `false`,
  unset `agentTool` texts and empty lists, and any sub-block whose `enabled` is `false`. A process
  not exposed to Factorial has no `factorial` key at all. A partial update
  leaves the unnamed fields unchanged.
- **Legacy `form` / `uiTrigger` keys still round-trip.** When a file carries
  both, `fcode push` sends `factorial` and lets the mirror update the rest.
  Prefer writing `factorial` (below).

## The `factorial` block, field by field

```json
{
  "factorial": {
    "enabled": true,
    "awaitResult": false,
    "requiredPolicies": [["company.manage_timeoff"], ["company.admin"]],
    "uiTrigger": { "enabled": true, "locationId": "calendar.header.admin", "label": "fcode.i18n(\"acme.approve.button\")", "icon": "Bell" },
    "form": { "enabled": true, "public": false, "appTool": true },
    "agentTool": { "enabled": true, "description": "…", "effect": "WRITE" },
    "backend": { "enabled": true }
  }
}
```

| Field | Type / default | Meaning |
|---|---|---|
| `enabled` | boolean, `false` | Master switch for every entry point below |
| `awaitResult` | boolean, `true` | `true`: Factorial waits and receives the `{ data }` / `{ errors }` envelope (`fcode-ui-triggers`). `false`: Factorial only learns the execution **started**; the process reports back itself (FactorialClient, notification). Use `false` for imports and bulk work |
| `requiredPolicies` | `string[][]`, `[]` | **OR of AND-groups** of Factorial policy keys the user must hold: `[["a","b"],["c"]]` reads *(a and b) or c*. Empty = anyone who reaches the entry point. Console: one alternative per line, commas between the keys of a group |
| `uiTrigger.enabled` | boolean | Render the action as a button inside Factorial |
| `uiTrigger.locationId` | string | Factorial UI location the button renders at (e.g. `calendar.header.admin`). **Required while enabled**, ≤ 200 chars, given by the Factorial team owning the page — see `fcode-ui-triggers` for the location contract |
| `uiTrigger.label` | string | Button text. **The only i18n field**: plain text or `fcode.i18n("key")` tokens |
| `uiTrigger.icon` | string | Allowlisted icon name (`fcode-ui-triggers`); omit for a text-only button |
| `form.enabled` | boolean | Expose the parameters as a form (`fcode-forms`). Same switch and same quota as the Forms flag |
| `form.appTool` | boolean, `false` | Offer the form to the company's users **on the App's own page** in Factorial, among the things the App lets them run — a "Report a sync issue" or "Sync now" form. Leave it `false` for a form the App opens itself. This is the block's name for the legacy `form.appRole: USER_FACING_FORM`, which it is kept mirrored with |
| `form.public` | boolean, `false` | **Deprecated.** Declares the form may *also* be embedded anonymously outside Factorial, as the legacy embed does. Stored, not enforced yet; set it only on forms that genuinely need anonymous access |
| `agentTool.enabled` | boolean | Let the Factorial One assistant call the process as a tool. The rest of the sub-block is the tool contract (next section) |
| `backend.enabled` | boolean | Let Factorial backend jobs run the process with no user in front of it |

## The Factorial One tool contract (`agentTool`)

The parameters schema tells the assistant **what the tool takes**; this block
tells it **what the tool is for**. Plain text, no i18n.

| Field | Type | What it buys the model |
|---|---|---|
| `description` | string | What the tool does, one or two sentences: name the object it acts on and the outcome. This is what user requests are matched against |
| `effect` | `READ` \| `WRITE` \| `DESTRUCTIVE` | `READ` runs without asking; `WRITE` asks for approval; `DESTRUCTIVE` asks and warns it cannot be undone. **Unset ⇒ `DESTRUCTIVE`** |
| `whenToUse` | string | The situation the tool is meant for, phrased as a user would ask |
| `whenNotToUse` | string[] | Look-alike situations that are *not* a match, one per entry — name the neighbour tool to use instead |
| `preconditions` | string[] | What must already be true before calling, one per entry; the assistant checks or asks instead of calling and failing |
| `doesNotDo` | string[] | Side effects a user might assume and the tool does not have, one per entry |
| `degradation` | string | What happens on partial success: what is done, what is skipped, how the result says so |
| `simulatesFor` | string | Slug of the process this one **previews** (a dry run of it). Free text, not validated yet |

### Writing good contract texts

- **Plain language, no i18n, one language.** Write for a model reading English,
  not for a screen. Never put `fcode.i18n(...)` tokens here.
- **`description`: object + outcome.** "Approves every pending time-off request
  of a team" beats "Time-off tool".
- **One situation in `whenToUse`.** If you need "or", you probably have two
  tools. Phrase it the way the user would ask.
- **Name neighbour tools in `whenNotToUse`.** "The user wants a single request
  approved: use `approve-timeoff-request`." Each entry is one situation.
- **`preconditions` are checkable facts** ("the team has at least one pending
  request"), not advice.
- **`doesNotDo` lists what a user would assume.** Notifications, side writes,
  undo — say what is *not* happening so the assistant can offer the missing
  step.
- **`degradation` describes the partial outcome**, so the assistant can explain
  it instead of reporting a failure.
- **A dry-run process points at its real one with `simulatesFor`** and is
  itself `READ`; the real one is `WRITE` or `DESTRUCTIVE`.

## Reserved lifecycle slugs

| Slug | Role | What the platform does |
|---|---|---|
| `install` | The form Factorial shows when a company installs the App | Derives `lifecycleRole: INSTALL` |
| `settings` | The form an installed company uses to configure the App | `SETTINGS` |
| `uninstall` | Runs when the company removes the App | `UNINSTALL` |
| `sync` | The sync process of an integrations-framework App | `SYNC` |

`lifecycleRole` is **read-only and derived from the slug**: the console shows a
tinted **Install / Settings / Uninstall / Sync** tag in the processes list and
the process header, the CLI and API report it, nothing ever writes it (it is not
in `metadata.json` and not in the content hash). `install`, `settings` and
`uninstall` are the marketplace lifecycle forms; they still take
`factorial.form.enabled` like any form. The pre-render pattern for re-opened
install/settings forms is in `fcode-forms`.

## Parameters: the contract with Factorial

Whatever the entry point, what Factorial sends is described by the process's
`parametersSchema.json` (`fcode-json-schema`). The form renders it, the
assistant reads it to know what the tool takes, Factorial validates against it.
Give every parameter a `title` and a `description`, list the mandatory ones in
`required`, and treat any schema change as a change to **what Factorial may
call you with**. Trusted keys a UI trigger injects (`company_id`,
`triggered_from_location`, `access_id`) are described in `fcode-ui-triggers`.

## The transition from `form` and `uiTrigger`

The `factorial` block supersedes the **Forms** flag (`form`) and the **UI
trigger** button (`uiTrigger`). Both stay for now and the platform keeps them
**mirrored**: whichever side you write, the other is updated, last write wins.

| Writing `factorial`… | …updates the legacy setting |
|---|---|
| `enabled && uiTrigger.enabled` | `uiTrigger.enabled` |
| `uiTrigger.locationId`, `uiTrigger.icon`, `uiTrigger.label` | the same keys of `uiTrigger` |
| `awaitResult` | `uiTrigger.awaitResult` |
| `enabled && form.enabled` | `form.enabled` |
| `form.appTool` | `form.appRole` = `USER_FACING_FORM` |

Writing the legacy blocks mirrors the same fields back, and `factorial.enabled`
turns on as soon as any entry point is enabled.

`form.appRole` is the one legacy field the block **partly** owns. Of its four
values, `INSTALL` / `SETTINGS` / `UNINSTALL` come from the reserved slugs and
are left untouched, while `USER_FACING_FORM` is `form.appTool` — turning the
flag on claims that value, turning it off releases it only if it was held, and
a write that says nothing about `appTool` leaves the role alone. So a file
written before the flag existed keeps its role on push: the CLI reads
`appRole` and sends the flag to match.

**Prefer `factorial`**: write it in new processes, and when touching an old
`metadata.json`, move the settings into it rather than editing `uiTrigger`.

## Worked example — SILTRA FIE import (dry run + apply)

Adapted from the `siltra-integration-app`: `fie-import` reads the `.msj` file
a company downloads from SILTRA and **reports** which sick leaves it would
create, link or close in Factorial, writing nothing; `fie-import-apply` does the
writing, in the background.

`processes/fie-import/metadata.json`:

```json
{
  "name": "FIE Import",
  "description": "Reads the .msj file the company downloads from SILTRA and shows what importing it would change in Factorial. Writes nothing — the import only happens after confirmation.",
  "tags": ["inbound", "siltra", "fie"],
  "factorial": {
    "enabled": true,
    "requiredPolicies": [["company.manage_timeoff"], ["company.admin"]],
    "uiTrigger": {
      "enabled": true,
      "locationId": "calendar.header.admin",
      "label": "fcode.i18n(\"calendar.import.fie.button\")",
      "icon": "Upload"
    },
    "form": { "enabled": true },
    "agentTool": {
      "enabled": true,
      "description": "Reads a SILTRA FIE transmission (.msj) and reports which sick leaves it would create, link or close in Factorial, and which events it would leave out and why. It writes nothing.",
      "effect": "READ",
      "whenToUse": "The user has downloaded a FIE file from SILTRA and wants to know what importing it would change before anything is written.",
      "whenNotToUse": [
        "The user has already reviewed the report and wants the leaves written: use fie-import-apply.",
        "The file is not a SILTRA .msj transmission."
      ],
      "preconditions": ["The .msj file has been uploaded and its storage reference is passed as msj_file."],
      "doesNotDo": ["It does not create, change or close any leave.", "It does not notify employees or managers."],
      "degradation": "If Factorial cannot be consulted, the report still lists every event in the file, marked as not compared against Factorial.",
      "simulatesFor": "fie-import-apply"
    }
  }
}
```

`processes/fie-import-apply/metadata.json` — no button, no form, asynchronous,
and the write side of the pair:

```json
{
  "name": "FIE Import — apply",
  "tags": ["inbound", "siltra", "fie"],
  "factorial": {
    "enabled": true,
    "awaitResult": false,
    "requiredPolicies": [["company.manage_timeoff"], ["company.admin"]],
    "agentTool": {
      "enabled": true,
      "description": "Writes the sick leaves a SILTRA FIE transmission describes into Factorial: creates, links or closes them.",
      "effect": "WRITE",
      "whenToUse": "The user has seen the fie-import report for a file and asks to apply it.",
      "whenNotToUse": ["The user only wants to know what the file would change: use fie-import."],
      "preconditions": ["fie-import has reported on this same file and the user confirmed the result."],
      "doesNotDo": ["It does not send SILTRA anything back.", "It does not notify employees."],
      "degradation": "Leaves that fail to write are skipped and listed in the result; the rest are applied."
    },
    "backend": { "enabled": true }
  }
}
```

The policy keys are illustrative — take the real ones from Factorial's policy
catalogue. Read the parameters as usual (`fcode.context.parameters.msj_file`,
an `fcode.storage://…` reference — `fcode-forms`).

## Checklist before shipping

1. `factorial.enabled` on, and only the entry points you mean to expose enabled.
2. `awaitResult` decided explicitly: `false` for anything slow; a synchronous
   action returns `{ data }` / `{ errors }`.
3. `requiredPolicies` keys copied from Factorial's catalogue, not guessed.
4. `agentTool`: `effect` set, `whenNotToUse` names the neighbour tools, dry runs
   point at their real process with `simulatesFor`.
5. No ordinary action uses `install`, `settings`, `uninstall` or `sync` as slug;
   the schema has titles, descriptions and `required`.
