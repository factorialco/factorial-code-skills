# Reference: marketplace integration ("Acme Payroll")

A complete outbound marketplace integration for an invented vendor, **Acme
Payroll**. Factorial triggers it per sync run; it delivers compensations and
leaves to the external system and reports a status per item back to Factorial.
This is the starting shape for any real payroll/HR integration.

## Architecture

```
Factorial ──webhook──▶ processes/sync (entry point)
                          │  validates challenge, delegates to the sync class
                          ▼
              modules/sync-file  OR  modules/sync-api   (pick ONE flavor)
                          │ extends
                          ▼
              outbound-sync (inherited from base-integration-app)
                          │ fetches syncable items, loops, reports statuses
                          ▼
              Factorial (per-item status: success / invalid / failed)
```

Workspace layout:

```
processes/sync/           # webhook entry point (form disabled)
modules/sync-file/        # flavor A: aggregate items → CSV → uploadOutput()
modules/sync-api/         # flavor B: push each item to the vendor HTTP API
modules/acme-client/      # thin vendor HTTP client (flavor B only)
```

## The `OutboundSync` contract (inherited, never reimplement)

`fcode.import("outbound-sync")` provides the orchestrator. Your integration
extends it:

**Must override**

- `get externalSystem()` — lowercase vendor slug, e.g. `"acme"`
- `get dataType()` — lowercase data slug, e.g. `"compensations"`
- `async process(syncableItem)` — called once per syncable item

**Optional hooks** (for batch/file-oriented targets)

- `async beforeSync(syncableItems)` — once, before the per-item loop (init
  clients, aggregation state)
- `async afterSync(syncableItems, results, parameters)` — once, after the
  loop; may mutate `results` in place before statuses are reported

**Provided helpers**

- `datastoreKey(key)` → `"externalSystem.dataType.key"`
- `integrationDatastoreKey(key)` → `"externalSystem.key"`
- `variableName(key)` → `"EXTERNALSYSTEM__KEY"`
- `uploadOutput({ content, syncRunId, prefix, extension, contentType })` —
  uploads a generated file as a sync run output via the SDK
  (`integrations.syncRunOutputs.create`); throws on non-2xx

**Entry point (never override)**: `async run(parameters)` — fetches the run's
syncable items, calls the hooks and `process()` per item, and reports one
status per item back to Factorial.

## Entry process — `processes/sync/index.js`

Factorial calls the webhook with `{ sync_run_id, integration_uuid, company_id }`
(declared in `parametersSchema.json` with `form.enabled: false` in
`metadata.json`). The process body stays minimal:

```javascript
const { checkWebhookChallenge } = fcode.import("factorial-utils");
const AcmeSync = fcode.import("sync-file"); // ← the ONE line that picks the flavor

async function main() {
  // Enforced only when FACTORIAL_CHALLENGE_TOKEN is configured.
  const challengeError = checkWebhookChallenge();
  if (challengeError) return challengeError;

  const sync = new AcmeSync();
  return await sync.run(fcode.context.parameters);
}

module.exports = { main };
```

## Pick ONE delivery flavor

| Flavor | Module | Delivery | Use when |
|---|---|---|---|
| **File export** | `sync-file` | Aggregate items into CSV files, upload via inherited `uploadOutput()` | Target system ingests files |
| **API push** | `sync-api` + `acme-client` | One HTTP request per item to the vendor API | Target system has a live API |

Build only the flavor the vendor needs; delete the other.

### Flavor A — file export (`modules/sync-file/sync-file.js`)

Accumulate rows in `beforeSync`/`process`, build and upload the file in
`afterSync`. Track which result slots each file owns so an upload failure marks
only the affected items failed:

```javascript
const OutboundSync = fcode.import("outbound-sync");

class AcmeFileSync extends OutboundSync {
  get externalSystem() { return "acme"; }
  get dataType() { return "compensations"; }

  async beforeSync() {
    this.compensationsRows = [];
    this.compensationsIndices = []; // result slots this file owns
    this.itemIndex = 0;
  }

  async process(syncableItem) {
    const idx = this.itemIndex++;
    const { syncable_type, sync_payload } = syncableItem;

    if (syncable_type === "compensations/compensation") {
      this.compensationsRows.push(mapCompensationRow(sync_payload));
      this.compensationsIndices.push(idx);
    } else {
      // Unsupported type = configuration problem, not a transient failure.
      const err = new Error(`Unsupported syncable_type: ${syncable_type}`);
      err.syncStatus = "invalid";
      throw err;
    }
  }

  async afterSync(syncableItems, results, parameters) {
    if (this.compensationsRows.length === 0) return;
    try {
      await this.uploadOutput({
        content: buildCsv(HEADER, this.compensationsRows),
        syncRunId: parameters.sync_run_id,
        prefix: "acme-compensations",
        extension: "csv",
        contentType: "text/csv",
      });
    } catch (err) {
      // File not delivered → every item in this file's group failed.
      for (const idx of this.compensationsIndices) {
        results[idx].status = "failed";
        results[idx].error_messages = { sync_api_error: err.message };
      }
    }
  }
}

module.exports = AcmeFileSync;
```

Mapping notes from the sample:

- Compensation `amount` arrives as **minutes** when `unit === "time"`
  (`amount / 60` → hours) and **cents** when `unit === "money"`
  (`amount / 100`).
- A leave payload is a date **range** (`starts_on`..`ends_on`, inclusive);
  the file flavor explodes it into one row per calendar day, iterating in UTC
  to avoid DST drift. Skip when `deleted_at` is set or `leave_type_code` is
  empty — return no rows, which counts as `success`.

### Flavor B — API push (`modules/sync-api/sync-api.js` + `modules/acme-client/`)

A thin dependency-free vendor client (global `fetch`, Node 22) plus a sync
class that pushes per item. The client reads credentials from integration
Team Variables and fails fast if missing:

```javascript
// modules/acme-client/acme-client.js
function createAcmeClient() {
  const baseUrl = process.env.ACME__API_BASE_URL;
  const apiKey = process.env.ACME__API_KEY;
  if (!baseUrl) throw new Error("Missing ACME__API_BASE_URL team variable.");
  if (!apiKey) throw new Error("Missing ACME__API_KEY team variable.");

  async function post(path, record) {
    const response = await fetch(`${baseUrl.replace(/\/+$/, "")}${path}`, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey}` },
      body: JSON.stringify(record),
    });
    if (response.ok) {
      const text = await response.text();
      return text ? JSON.parse(text) : {};
    }
    const err = new Error(`Acme API POST ${path} failed (status ${response.status})`);
    if (response.status === 400 || response.status === 422) {
      err.isValidationError = true; // → reported "invalid", not "failed"
    }
    throw err;
  }

  return { pushCompensation: (r) => post("/compensations", r) };
}
```

```javascript
// modules/sync-api/sync-api.js
class AcmeApiSync extends OutboundSync {
  get externalSystem() { return "acme"; }
  get dataType() { return "compensations"; }

  async beforeSync() {
    // Fail fast on missing credentials, before the per-item loop.
    this.client = createAcmeClient();
  }

  async process(syncableItem) {
    const record = mapCompensationRecord(syncableItem.sync_payload);
    try {
      await this.client.pushCompensation(record);
    } catch (err) {
      if (err.isValidationError) err.syncStatus = "invalid";
      throw err; // anything else → "failed"
    }
  }
}
```

## Status semantics

`OutboundSync.run()` reports one status per item:

| Status | Meaning | How to signal |
|---|---|---|
| `success` | Delivered, or intentionally skipped (e.g. deleted leave) | `process()` returns normally |
| `invalid` | Configuration/validation error — retry won't help | throw with `err.syncStatus = "invalid"` |
| `failed` | Transient error (network, 5xx, upload failure) — retryable | throw anything else |

## Required Team Variables

| Variable | Used by | Notes |
|---|---|---|
| `FACTORIAL_TOKEN` | both flavors | Authenticates the Factorial SDK client |
| `FACTORIAL_BASE_URL` | both | Optional API base URL override |
| `FACTORIAL_CHALLENGE_TOKEN` | both | Optional; when set, the webhook challenge is enforced |
| `ACME__API_BASE_URL` | API flavor | Vendor API base URL |
| `ACME__API_KEY` | API flavor | Vendor bearer token (sensitive) |

Vendor variables follow the `VENDOR__KEY` convention (matches
`variableName()`).

## Adapting to a real vendor — checklist

1. Pick the flavor; set the one-line import in `processes/sync/index.js`.
2. Rename `externalSystem`, class names, file prefixes, and `ACME__*`
   variables to the vendor.
3. File flavor: rewrite the CSV layout (columns, separator, header — real
   vendor formats often omit the header row).
4. API flavor: rewrite endpoints, auth scheme, and record mapping in the
   client module.
5. Delete the unused flavor's modules.
6. Keep the status semantics: validation errors → `invalid`, transient
   errors → `failed`, skips → `success`.
