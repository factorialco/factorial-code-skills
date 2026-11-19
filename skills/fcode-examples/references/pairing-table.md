# Reference: pairing Factorial records with another system ("Acme ERP departments")

A settings form that maps each Factorial **team** to a **department** in a
fictional ERP, Acme. Instead of a dropdown per team it uses a pairing table
(`"ui:field": "pairing"`). The table pre-pairs teams whose names match, shows
the teams in use first, and lets the admin review and save. The pattern is the
same for any Factorial-to-vendor mapping: accounts to a chart of accounts, tax
codes, cost centers, wage types, absence codes or positions. Only the two lists
and the matching rule change. The contract (row shape, suggestions, in-use,
"not mapped", options) is owned by `fcode-forms` §Pairing tables; this file is
the code.

## Architecture

```
FORM   settings (preRenderProcess: settings-prerender)
         │ the pre-render lists Factorial teams + Acme departments, reads the
         │ saved pairs, proposes matches, and returns the whole section
         ▼
TABLE  Confirmed (saved) · Suggested (same name) · Needs a choice
         │ admin accepts / picks / clears, then Save
         ▼
SAVE   settings receives { departments: { team_12: "D-40", … } }
         │ checks every value against Acme, stores the map
         ▼
SYNC   later processes read the map from the datastore
```

Workspace layout:

```
processes/settings/             # the form; validates and stores the pairs
processes/settings-prerender/   # builds the pairing section
modules/acme-client/            # the vendor API client (adapt to the real vendor)
```

## The form — `processes/settings/parametersSchema.json`

The whole section comes from the pre-render, so the schema only holds a
fallback: an empty pairing object, used when the pre-render does not run.

```json
{
  "title": "Acme ERP settings",
  "type": "object",
  "preRenderProcess": "settings-prerender",
  "variables": {
    "departmentsField": {
      "type": "object",
      "title": "Departments",
      "ui": { "ui:field": "pairing" },
      "properties": {}
    }
  },
  "properties": {
    "departments": { "$ref": "#/variables/departmentsField" }
  }
}
```

## Pre-render — build the section

```javascript
// processes/settings-prerender/index.js
const MAP_KEY = "acme.departments.map";
const ROW_PREFIX = "team_";

const normalize = (name) => name.trim().toLowerCase().replace(/\s+/g, " ");

async function main() {
  const { createFactorialClient } = fcode.import("factorial-sdk");
  const { AcmeClient } = fcode.import("acme-client");

  // Teams stay in the tens or hundreds even for large accounts, which a pairing
  // table handles. Never pair every employee this way (fcode-json-schema §Gotchas).
  const teams = await createFactorialClient().teams.teams.all();

  // The form must open even when Acme can't be reached (no API key yet right
  // after install, Acme down): that is where the admin fixes the setup.
  let departments = [];
  let unreachable = false;
  try {
    departments = await new AcmeClient(fcode.env.ACME_API_KEY).listDepartments();
  } catch (error) {
    console.error("Acme departments unavailable", error);
    unreachable = true;
  }

  let saved = {};
  try {
    saved = JSON.parse((await fcode.datastore.get(MAP_KEY)) || "{}");
  } catch (error) {
    console.error(`Stored ${MAP_KEY} is not valid JSON; starting from an empty map`, error);
  }

  const choices = [
    { const: "", title: "Not mapped" },
    ...departments.map((d) => ({ const: d.code, title: `${d.code} · ${d.name}` })),
  ];
  const byName = new Map(departments.map((d) => [normalize(d.name), d]));

  const rows = Object.fromEntries(
    teams.map((team) => {
      const stored = saved[String(team.id)];
      const match = byName.get(normalize(team.name));
      const row = {
        type: "string",
        title: team.name,
        // The stored pair is the default: it shows as Confirmed. A stored ""
        // is the admin's explicit "Not mapped".
        ...(stored !== undefined ? { default: stored } : {}),
        // A proposal is never a default: it is submitted only once accepted.
        ...(stored === undefined && match
          ? { "x-fcode-suggestion": { value: match.code, reason: "Same name" } }
          : {}),
        // Teams with people are the ones worth reviewing first.
        ...((team.employee_ids ?? []).length > 0 ? { "x-fcode-in-use": true } : {}),
      };
      return [`${ROW_PREFIX}${team.id}`, row];
    })
  );

  return {
    variables: {
      departmentsField: {
        type: "object",
        title: "Departments",
        description: unreachable
          ? "Acme couldn't be reached, so its departments can't be listed. Check the API key and try again."
          : "Pair each Factorial team with its Acme department.",
        ui: {
          "ui:field": "pairing",
          "ui:options": {
            columns: ["Factorial team", "Acme department"],
            inUseNote:
              "Teams with people, and any team already paired or with a suggestion, are listed. Add any other team below.",
            labels: { addRow: "Add team" },
          },
        },
        // One list for the whole section: rows without their own choices use it.
        "x-fcode-choices": choices,
        properties: rows,
      },
    },
  };
}

module.exports = { main };
```

Notes on the pre-render:

- **`""` is the explicit "Not mapped" choice.** The admin picks it in the table,
  it is stored as `""`, and it comes back as the row's `default`, Confirmed.
  Never send `"default": ""` as a blank: a row with nothing stored has no
  `default`.
- **The suggestion only exists when nothing is stored.** A row with both shows
  as Confirmed anyway, but leaving it out keeps the schema honest for Factorial
  One, which reads `x-fcode-*` keys.
- **The choices are sent once, as the section's `x-fcode-choices`**, not on
  every row, so 300 teams and 300 departments stay a small schema. Keep the
  list to what can be paired (active departments only, say): a stored pair that
  falls out of it shows as **No longer available** and asks for a new choice.
- **Vendor and datastore errors are caught.** An error thrown out of a
  pre-render makes the form unopenable (`fcode-forms`, `references/advanced.md`),
  and the settings form is where the admin fixes a missing API key. With Acme
  unreachable, the section has no choices and its description says why.
- **The in-use list also shows paired, suggested and no-longer-available rows**,
  so the note says so: a team with no people but a matching name is listed too.

## Save — `processes/settings/index.js`

```javascript
const MAP_KEY = "acme.departments.map";
const ROW_PREFIX = "team_";

async function main() {
  const { AcmeClient } = fcode.import("acme-client");
  const submitted = fcode.context.parameters.departments ?? {};

  // Parameters are untrusted: check every value against the vendor's list.
  const known = new Set(
    (await new AcmeClient(fcode.env.ACME_API_KEY).listDepartments()).map((d) => d.code)
  );
  const map = {};
  for (const [key, code] of Object.entries(submitted)) {
    if (!key.startsWith(ROW_PREFIX) || typeof code !== "string") continue;
    // "" is the admin's explicit "Not mapped": stored like any other choice.
    if (code !== "" && !known.has(code)) {
      return {
        status: 400,
        body: { errorMessage: `Unknown Acme department **${code}**. Reload the form and try again.` },
      };
    }
    map[key.slice(ROW_PREFIX.length)] = code;
  }

  await fcode.datastore.set(MAP_KEY, JSON.stringify(map));
  return { message: `Saved ${Object.keys(map).length} department pairs.` };
}

module.exports = { main };
```

The submission is a flat `{ team_12: "D-40", team_13: "" }` object, the same
shape as a section of plain dropdowns: confirmed rows carry their code, rows
set to "Not mapped" carry `""`, and rows cleared or never paired (including
suggestions nobody accepted) are absent. Rebuilding the map from the submission
alone therefore stores exactly what was reviewed.

## Adapting it

- **Another vendor or entity**: swap `teams.teams.all()` for the Factorial
  records being mapped, and `listDepartments()` for the vendor's list. Keep the
  row key prefixed so ids that start with a digit stay valid keys.
- **Better matching**: compute `x-fcode-suggestion` from whatever the app knows
  (same code, same number, a previous customer's choice) and say why in
  `reason`. The admin sees it on hover over the suggested row.
- **Grouping**: put several pairing sections in one form (one per legal entity,
  say), each its own `object` with `"ui:field": "pairing"`.
