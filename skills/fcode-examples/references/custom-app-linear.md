# Reference: custom app with lifecycle ("Linear integration")

A complete custom app that connects Factorial to **Linear**: a two-step setup
form, a webhook-driven outbound push, a scheduled inbound poll, and a clean
uninstall. This is the pattern for any app that must be *installed* by an
end user (collect credentials, map entities, create webhooks/schedules) and
later *uninstalled* without leaving residue.

## Architecture

```
SETUP (user-facing forms)
  linear-setup ──nextProcessId──▶ linear-setup-mapping
       │                              │ form rendered with data from
       │ stores LINEAR_API_KEY        │ linear-setup-mapping-prerender
       ▼                              ▼
                            stores mapping, creates webhook + schedule,
                            records their ids in the datastore

SETTINGS (linear-setup-mapping re-opened after install)
  its pre-render reads variables + datastore so the form shows the current
  configuration; a credential is reported as set, never echoed back

RUNTIME
  linear-users-push     ◀── Factorial webhook (employee created)
  linear-projects-poll  ◀── hourly schedule (created at setup)

TEARDOWN
  linear-uninstall — deletes webhook, schedule, variable, datastore keys
```

Workspace layout:

```
processes/linear-setup/                    # step 1: validate + store API key
processes/linear-setup-mapping/            # step 2: map teams, activate
processes/linear-setup-mapping-prerender/  # renders step 2's dropdowns + current values
processes/linear-users-push/               # webhook: Factorial employee → Linear user
processes/linear-projects-poll/            # schedule: Linear projects → Factorial
processes/linear-uninstall/                # teardown
modules/linear-client/                     # vendor API client (wraps @linear/sdk)
```

## Triggers per process — `metadata.json`

Each process declares how it's invoked in its `metadata.json` (full field
reference in `fcode-cli`). The setup entry point is a form and carries the
app's `INSTALL` role; the runtime push process is webhook-only:

`processes/linear-setup/metadata.json` — the app's install form:

```json
{
  "name": "Connect Linear",
  "tags": ["linear", "setup"],
  "form": { "enabled": true, "authMode": "FACTORIAL", "appRole": "INSTALL" }
}
```

Both setup forms are opened from inside Factorial, so they carry
`authMode: FACTORIAL` — only Factorial users of the company that installed the
app can read the schema or submit it. A public app form would let anyone who
knows the workspace and process slugs run app code against customer data.

`processes/linear-setup-mapping/metadata.json` — step 2 of install, reached
via `nextProcessId`. It also doubles as the app's post-install settings
screen (re-map teams later), which is what `appRole: SETTINGS` marks — an
app has at most one `INSTALL` and one `SETTINGS` process:

```json
{
  "name": "Map Linear teams",
  "tags": ["linear", "setup"],
  "form": { "enabled": true, "authMode": "FACTORIAL", "appRole": "SETTINGS" }
}
```

`processes/linear-users-push/metadata.json` — webhook target, never a form:

```json
{
  "name": "Linear users push",
  "tags": ["linear"],
  "webhook": { "enabled": true, "authMode": "TEAM" },
  "form": { "enabled": false }
}
```

`authMode: TEAM` inherits the workspace `webhookAuth` from `team.json`, which for a
Factorial-sent webhook expects `FACTORIAL_CHALLENGE_TOKEN` in the
`x-factorial-wh-challenge` header — the same token `setupWebhook` embeds in the
subscription at install (see below). The platform checks it before the process
runs, so webhook processes carry no auth code. Set `webhookAuth` per workspace; it
is not inherited from `parentTeamSlugs`. See `fcode-cli`.

`linear-projects-poll` needs neither webhook nor form — it's invoked by the
schedule created at install time. Edit these files and `fcode push`; no
dashboard clicking needed.

## The vendor client module

One module encapsulates every vendor call; processes never touch the vendor
SDK directly. Dependencies are declared inline:

```javascript
// modules/linear-client/linear-client.js
// @add-package @linear/sdk
const { LinearClient } = require("@linear/sdk");

class LinearApiClient {
  constructor(apiKey) {
    if (!apiKey) throw new Error("Linear API key is required");
    this.client = new LinearClient({ apiKey });
  }
  async userExists(email) { /* ... */ }
  async updateUser(userData) { /* ... */ }
  async inviteUser(userData) { /* ... */ }
  async getProjects({ includeArchived = false, updatedAfter = null } = {}) { /* ... */ }
  async getTeams() { /* ... */ }
}

module.exports = { LinearApiClient };
```

## Setup step 1 — validate credentials, chain to step 2

A minimal form (one credential field with `"isSensitive": true`, which renders
a password input — see `fcode-json-schema`). The process validates the key
with a real API call, persists it as a team variable, and chains by returning
`nextProcessId`:

```javascript
async function main() {
  const { linear_api_key } = fcode.context.parameters;
  if (!linear_api_key) throw new Error("linear_api_key is required");

  // 1. Validate by making a real call.
  const { LinearApiClient } = fcode.import("linear-client");
  try {
    await new LinearApiClient(linear_api_key).getTeams();
  } catch (error) {
    throw new Error(`Invalid Linear API key or network error: ${error.message}`);
  }

  // 2. Persist for the pre-render and runtime processes (sensitive by default).
  await fcode.variables.set("LINEAR_API_KEY", linear_api_key);

  // 3. Chain to step 2 by slug — slugs are identical in every workspace.
  return { nextProcessId: "linear-setup-mapping" };
}
```

Returning `{ nextProcessId }` is what makes the form multi-step: after
submitting step 1, the UI renders `nextProcessId`'s form as step 2.

## Setup step 2 — dynamic dropdowns via `preRenderProcess`

Step 2's form maps each Linear team to a Factorial team. The dropdown options
don't exist at authoring time — they're produced at **render time** by a
helper process wired through the schema's `preRenderProcess` attribute.

The schema `$ref`s nodes under `#/variables`; the pre-render process returns
those nodes:

```json
{
  "title": "Map Linear teams to Factorial teams",
  "type": "object",
  "preRenderProcess": "linear-setup-mapping-prerender",
  "variables": {
    "linearTeamField": { "title": "Linear team", "type": "string" },
    "factorialTeamField": { "title": "Factorial team", "type": "string" },
    "teamMappingsScaffold": []
  },
  "properties": {
    "team_mappings": {
      "type": "array",
      "minItems": 1,
      "default": { "$ref": "#/variables/teamMappingsScaffold" },
      "items": {
        "type": "object",
        "properties": {
          "linear_team_id": { "$ref": "#/variables/linearTeamField" },
          "factorial_team_id": { "$ref": "#/variables/factorialTeamField" }
        },
        "required": ["linear_team_id", "factorial_team_id"]
      }
    }
  },
  "required": ["team_mappings"]
}
```

```javascript
// processes/linear-setup-mapping-prerender/index.js — runs while the form is served
async function main() {
  const { selectField, toOptions } = fcode.import("fcode-forms");

  const linearApiKey = process.env.LINEAR_API_KEY; // stored by step 1
  if (!linearApiKey) throw new Error("LINEAR_API_KEY is not set. Run linear-setup first.");

  const { LinearApiClient } = fcode.import("linear-client");
  const linearTeams = (await new LinearApiClient(linearApiKey).getTeams())
    .map((t) => ({ id: t.id, name: t.name }));

  const { createFactorialClient } = fcode.import("factorial-sdk");
  const factorialTeams = (await createFactorialClient().teams.teams.all())
    .map((t) => ({ id: t.id, name: t.name }));

  // The same process backs appRole: SETTINGS, so the form is re-opened after
  // install — scaffold the rows from the mapping already saved, not blank ones.
  const saved = JSON.parse((await fcode.datastore.get("linear.teams.mappings")) || "{}");

  return {
    // Merged into the form schema's #/variables before rendering.
    variables: {
      linearTeamField: selectField("Linear team", "Linear team.", toOptions(linearTeams)),
      factorialTeamField: selectField("Factorial team", "Maps to.", toOptions(factorialTeams)),
      // One row per Linear team, pre-filled with the current mapping where there
      // is one. On first install `saved` is {} and the rows come out unmapped.
      teamMappingsScaffold: linearTeams.map((t) => ({
        linear_team_id: String(t.id),
        ...(saved[String(t.id)] ? { factorial_team_id: saved[String(t.id)] } : {}),
      })),
    },
  };
}
```

See `fcode-forms` for the full pre-render contract.

## Setup step 2 — activate: webhook + schedule + install records

The activation process persists the mapping and creates the runtime plumbing.
**Everything it creates is recorded in the datastore so uninstall can find
it later** — this is the core lifecycle discipline:

```javascript
const MAPPINGS_KEY = "linear.teams.mappings";
const SUBSCRIPTIONS_KEY = "linear.install.subscriptions";
const SCHEDULES_KEY = "linear.install.schedules";

async function main() {
  const { team_mappings } = fcode.context.parameters;

  // 1. Persist the mapping (datastore = strings only → JSON.stringify).
  const mapping = {};
  for (const row of team_mappings) {
    mapping[String(row.linear_team_id)] = String(row.factorial_team_id);
  }
  await fcode.datastore.set(MAPPINGS_KEY, JSON.stringify(mapping));

  // 2. Create the Factorial webhook (employee created → linear-users-push).
  const { createFactorialClient } = fcode.import("factorial-sdk");
  const { getCompanyId, setupWebhook } = fcode.import("factorial-utils");
  const factorialClient = createFactorialClient();
  const companyId = await getCompanyId(factorialClient);

  // setupWebhook embeds FACTORIAL_CHALLENGE_TOKEN as the subscription's challenge — the
  // value Factorial then sends in x-factorial-wh-challenge and the workspace webhookAuth
  // checks. It throws when the variable is unset, so the endpoint is never left open.
  const subscription = await setupWebhook({
    factorialClient,
    subscriptionType: "employees/employee/create_with_contract",
    processSlug: "linear-users-push",
    companyId,
    name: `${fcode.team.slug}-linear-users-push`,
  });
  // Gotcha: the API response field is "type", not "subscription_type".
  const subscriptionType = subscription.type ?? subscription.subscription_type;
  await fcode.datastore.set(SUBSCRIPTIONS_KEY,
    JSON.stringify([{ id: subscription.id, subscription_type: subscriptionType }]));

  // 3. Schedule the hourly poll; record its id.
  const created = await fcode.schedule.create("linear-projects-poll", {
    cron: "0 0 */1 * * *",
    allowConcurrentExecutions: false,
  });
  await fcode.datastore.set(SCHEDULES_KEY,
    JSON.stringify([{ id: created.id, process_slug: "linear-projects-poll" }]));

  return { success: true };
}
```

## Re-opened as the settings screen — show what is configured now

`linear-setup-mapping` carries `appRole: SETTINGS`, so the *same* form is opened
again long after install, to re-map teams or rotate the key. **It must render the
current state**, or the user is asked to re-enter configuration they cannot see —
and an app has at most one `SETTINGS` process, so this is that form, not a second
one.

The pre-render above already does this for the mapping. Extending it to cover the
credential and a poll interval is the whole pattern:

```javascript
// processes/linear-setup-mapping-prerender/index.js (continued)
const MAPPINGS_KEY = "linear.teams.mappings";
const SETTINGS_KEY = "linear.settings";

// …linearTeams / factorialTeams loaded as above…

// Never read a secret's value to echo it — only test whether one exists.
// process.env sees inherited variables too, so the key may be configured in a
// parent workspace (see fcode-core-concepts) and still count as configured.
const apiKeyConfigured = Boolean(process.env.LINEAR_API_KEY);

// Default missing state rather than throwing: a pre-render that throws makes the
// settings form unopenable, which is exactly when it is needed most.
const settings = JSON.parse((await fcode.datastore.get(SETTINGS_KEY)) || "{}");
const mapping = JSON.parse((await fcode.datastore.get(MAPPINGS_KEY)) || "{}");

return {
  variables: {
    linearTeamField: /* … as above … */,
    factorialTeamField: /* … as above … */,
    teamMappingsScaffold: /* … pre-filled from `mapping`, as above … */,

    // Plain config round-trips straight into the field default.
    pollIntervalDefault: settings.pollIntervalHours ?? 1,
    // The credential is reported, not returned.
    apiKeyLabel: apiKeyConfigured
      ? "Linear API key (configured — leave blank to keep the current one)"
      : "Linear API key",
    statusHtml: {
      before: apiKeyConfigured
        ? `<p>Connected. ${Object.keys(mapping).length} team(s) mapped.</p>`
        : "<p><b>Not connected yet.</b> Enter an API key to finish setup.</p>",
    },
  },
};
```

The schema points its `default`s, its label and a status line at those nodes. Only
the added nodes are shown below — `team_mappings` and its variables from the
previous section stay as they are. The API-key field is **not** `required` —
otherwise a user who only wants to re-map a team cannot submit:

```json
{
  "preRenderProcess": "linear-setup-mapping-prerender",
  "variables": {
    "pollIntervalDefault": 1,
    "apiKeyLabel": "Linear API key",
    "statusHtml": {}
  },
  "properties": {
    "linear_api_key": {
      "type": "string",
      "isSensitive": true,
      "title": { "$ref": "#/variables/apiKeyLabel" },
      "rawHtml": { "$ref": "#/variables/statusHtml" }
    },
    "poll_interval_hours": {
      "type": "integer",
      "title": "Poll every (hours)",
      "default": { "$ref": "#/variables/pollIntervalDefault" }
    }
  }
}
```

The activation process then treats a blank credential as "unchanged", which is
what makes leaving it optional safe:

```javascript
const { linear_api_key, poll_interval_hours } = fcode.context.parameters;

// Blank → keep the stored key. Never overwrite a working credential with "".
if (linear_api_key) {
  const { LinearApiClient } = fcode.import("linear-client");
  await new LinearApiClient(linear_api_key).getTeams();        // validate before storing
  await fcode.variables.set("LINEAR_API_KEY", linear_api_key); // rotate
}

await fcode.datastore.set(
  SETTINGS_KEY,
  JSON.stringify({ pollIntervalHours: poll_interval_hours })
);
```

The rules this encodes — a secret is reported, never echoed; blank on submit
means "keep the current value"; the credential field is not `required`; a
pre-render defaults missing state instead of throwing — are the pre-fill
contract in `fcode-forms` (`references/advanced.md`).

The no-throw rule changes the install-flow snippet above. Throwing on a missing
`LINEAR_API_KEY` is fine while the form is only ever reached from step 1, which
just stored it — but the moment the same form is also the `SETTINGS` screen, that
throw makes it unopenable in precisely the state where the user needs it to enter
a key. Degrade instead:

```javascript
// Replaces `if (!linearApiKey) throw ...` once the form doubles as SETTINGS.
const linearTeams = apiKeyConfigured
  ? (await new LinearApiClient(process.env.LINEAR_API_KEY).getTeams())
      .map((t) => ({ id: t.id, name: t.name }))
  : []; // no key yet → empty dropdowns, and statusHtml explains why

// Also guard the vendor call itself: an expired key must render the form with a
// "reconnect" status, not fail the load.
```

The rule of thumb: a pre-render backing an `INSTALL`-only form may throw, because
a broken install is a dead end anyway. A pre-render backing a `SETTINGS` form
should render *something* for every state, since fixing the broken state is what
the user came to do.

## Runtime — scheduled poll with cursor + idempotency

`linear-projects-poll` pulls Linear projects into Factorial. Two datastore
keys make it incremental and idempotent:

- a **cursor** (ISO timestamp of the last run) so each run only fetches
  what changed, and
- a **dedup map** (`{ linearProjectId: factorialProjectId }`) so re-delivered
  items are skipped.

```javascript
const CURSOR_KEY = "linear:projects:last_poll_timestamp";
const SYNCED_KEY = "linear.projects.synced";

async function main() {
  const lastPollTimestamp = await fcode.datastore.get(CURSOR_KEY); // null → full sync
  const projects = await linearClient.getProjects({ updatedAfter: lastPollTimestamp || null });
  const runTimestamp = new Date().toISOString();

  const synced = JSON.parse((await fcode.datastore.get(SYNCED_KEY)) || "{}");
  for (const project of projects) {
    if (synced[project.id]) continue; // already created
    const { data, error, response } = await factorialClient.projectManagement.projects.create({
      body: { name: project.name, code: project.id }, // code = vendor id, for traceability
    });
    if (error || (response && !response.ok)) { /* collect error, continue */ }
    synced[project.id] = (data && data.data ? data.data : data).id;
  }

  await fcode.datastore.set(SYNCED_KEY, JSON.stringify(synced));
  await fcode.datastore.set(CURSOR_KEY, runTimestamp);
}
```

Per-item failures are collected and returned, never allowed to abort the run.

The dedup map is needed here because the target (the Factorial API) has no
native upsert. **When the target API offers a server-side upsert, prefer it**
and drop the dedup state entirely — idempotency becomes a property of the API
call (e.g. Airtable's `PATCH` with `performUpsert.fieldsToMergeOn`, 1–3
non-computed fields). Keep the cursor either way; keep the dedup map only for
targets without upsert.

## Runtime — webhook push

`linear-users-push` receives the employee-created webhook and upserts the
user in Linear (update if the email exists, invite otherwise):

```javascript
async function main() {
  // The caller is already authenticated: the webhook's authMode: TEAM resolves to the
  // workspace webhookAuth, which checks FACTORIAL_CHALLENGE_TOKEN in
  // x-factorial-wh-challenge. Validate the payload, not the credential.
  const { id, login_email, full_name, preferred_name } = fcode.context.parameters;
  if (!login_email) {
    // Bad payload → webhook-style HTTP response, not a crash.
    return { status: 400, body: { error: { message: "Missing login_email" } } };
  }

  const linearClient = new LinearApiClient(process.env.LINEAR_API_KEY);
  if (await linearClient.userExists(login_email)) {
    await linearClient.updateUser({ email: login_email, full_name, preferred_name });
    return { action: "updated", email: login_email };
  }
  await linearClient.inviteUser({ email: login_email, full_name, preferred_name });
  return { action: "invited", email: login_email };
}
```

## Uninstall — best-effort teardown

Reverses setup using the install records. Every step is wrapped in its own
try/catch and logged, so a **partial** install can still be cleaned; a
`confirm` parameter guards against accidental runs:

```javascript
async function main() {
  const { confirm } = fcode.context.parameters;
  if (!confirm) throw new Error("Set confirm = true to uninstall the Linear integration");

  // 1. Delete webhooks recorded at install time.
  const { deleteWebhookSubscription } = fcode.import("factorial-utils");
  const subscriptions = JSON.parse((await fcode.datastore.get(SUBSCRIPTIONS_KEY)) || "[]");
  for (const sub of subscriptions) {
    try { await deleteWebhookSubscription(factorialClient, sub.id); }
    catch (error) { console.error(`Failed to delete webhook ${sub.id}: ${error.message}`); }
  }

  // 2. Delete schedules by recorded id.
  const schedules = JSON.parse((await fcode.datastore.get(SCHEDULES_KEY)) || "[]");
  for (const s of schedules) {
    try { await fcode.schedule.delete(s.id); } catch (error) { /* log, continue */ }
  }

  // 3. Delete the credential variable, then all app datastore keys.
  try { await fcode.variables.delete("LINEAR_API_KEY"); } catch (error) { /* log */ }
  for (const key of [MAPPINGS_KEY, SUBSCRIPTIONS_KEY, SCHEDULES_KEY, CURSOR_KEY]) {
    try { await fcode.datastore.del(key); } catch (error) { /* log */ }
  }

  return { success: true };
}
```

## Adapting to another vendor — checklist

1. Replace `linear-client` with a module wrapping the vendor's SDK/API
   (`@add-package` for the dependency).
2. Step 1: swap the credential field(s) and the validation call; keep the
   store-variable + `nextProcessId` chain.
3. Pre-render: load whatever collections the mapping needs (teams, projects,
   ledgers, …) and return them as `#/variables` nodes with
   `selectField`/`toOptions`, **plus the values already configured** so the form
   shows current state when re-opened as the settings screen — reporting whether
   a credential is set, never echoing it.
4. Step 2: keep the discipline — persist config, create webhooks/schedules,
   **record every created id in the datastore**.
5. Runtime processes: keep cursor + dedup for polling (vendor-native upsert
   instead of the dedup map when the target API has one); keep `authMode: TEAM` +
   HTTP-style error returns for webhooks.
6. Uninstall: delete exactly what the install records say, best-effort,
   behind a `confirm` flag.
