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
processes/linear-setup-mapping-prerender/  # renders step 2's dropdowns
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
  "form": { "enabled": true, "appRole": "INSTALL" }
}
```

`processes/linear-setup-mapping/metadata.json` — step 2, reached via
`nextProcessId`, so no role of its own:

```json
{
  "name": "Map Linear teams",
  "tags": ["linear", "setup"],
  "form": { "enabled": true }
}
```

`processes/linear-users-push/metadata.json` — webhook target, never a form:

```json
{
  "name": "Linear users push",
  "tags": ["linear"],
  "webhook": { "enabled": true },
  "form": { "enabled": false }
}
```

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

A minimal form (one password field, `"isSensitive": true`,
`"ui:widget": "password"`). The process validates the key with a real API
call, persists it as a team variable, and chains by returning
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

  // 3. Chain to step 2. The id is per-workspace config with a slug fallback.
  const nextProcessId =
    process.env.LINEAR_SETUP_MAPPING_PROCESS_ID || "linear-setup-mapping";
  return { nextProcessId };
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

  return {
    // Merged into the form schema's #/variables before rendering.
    variables: {
      linearTeamField: selectField("Linear team", "Linear team.", toOptions(linearTeams)),
      factorialTeamField: selectField("Factorial team", "Maps to.", toOptions(factorialTeams)),
      // One pre-filled row per Linear team.
      teamMappingsScaffold: linearTeams.map((t) => ({ linear_team_id: String(t.id) })),
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

## Runtime — webhook push

`linear-users-push` receives the employee-created webhook and upserts the
user in Linear (update if the email exists, invite otherwise):

```javascript
async function main() {
  const { checkWebhookChallenge } = fcode.import("factorial-utils");
  const challengeError = checkWebhookChallenge();
  if (challengeError) return challengeError;

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
   `selectField`/`toOptions`.
4. Step 2: keep the discipline — persist config, create webhooks/schedules,
   **record every created id in the datastore**.
5. Runtime processes: keep cursor + dedup for polling; keep challenge check +
   HTTP-style error returns for webhooks.
6. Uninstall: delete exactly what the install records say, best-effort,
   behind a `confirm` flag.
