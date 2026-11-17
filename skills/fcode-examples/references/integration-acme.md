# Reference: marketplace integration ("Acme Payroll")

A complete outbound marketplace integration for an invented vendor, **Acme
Payroll**. Factorial starts a sync run; the App reads the run's compensations
from the public API, delivers them to Acme, and reports a status per item back
to Factorial. This is the starting shape for any real payroll/HR integration.

## Architecture

```
Factorial ──apps/sync──▶ processes/sync          (inherited from base-integration-app)
                              │ fcode.import("sync")
                              ▼
                        modules/sync             (yours: the ONE module you ship)
                              │ class AcmeSync extends FileSync(CompensationsSync)
                              │                    or ApiSync(CompensationsSync)
                              ▼
                        record-sync              (inherited core)
                              │ reads the run, reads each record by id,
                              │ calls your hooks, reports per-item status
                              ▼
                        Factorial (per item: success / invalid / failed)
```

The integration is a **vertical** (what it syncs) composed with a **delivery
mode** (how it delivers), both inherited:

| Vertical | Module | `dataType` |
|---|---|---|
| `CompensationsSync` | `compensations-sync` | `compensations` (payroll-run compensations) |
| `ExpensesSync` | `expenses-sync` | `expenses` |
| `LeavesSync` | `leaves-sync` | `leaves` |
| `FinanceVendorSync` | `finance-vendor-sync` | `finance_vendors` |
| `FinancialDocumentSync` | `financial-document-sync` | `financial_documents` |

| Mode | Module | You implement | Use when |
|---|---|---|---|
| `FileSync` | `file-sync` | `mapRecord(item)` → rows, `render(rows)` → file | The vendor ingests files |
| `ApiSync` | `api-sync` | `process(item)` → deliver one item | The vendor has a live API |

Workspace layout:

```
modules/sync/              # your sync class — module.exports = the class itself
modules/acme-client/       # thin vendor HTTP client (API mode only)
```

That's all. **Don't add `processes/sync`, `processes/validate` or
`processes/digests`**: the base ships all three, and a process of your own
overrides the base's and stops the App from picking up its fixes.

## The inherited entry points (never reimplement)

- **`processes/sync`** — Factorial's monolith calls the dashboard's
  `apps/sync`, which relays `{ sync_run_id, integration_uuid, company_id }` to
  this process's webhook in the company's `deploy-` workspace. It does
  `new (fcode.import("sync"))().run(parameters)`. Your module must have the
  exact slug `sync`; the base defines none, so yours always wins.
- **`processes/validate`** — called before a run is created, while a review
  modal waits; it runs your optional `validate(item)` hook (below).
- **`processes/digests`** — lets Factorial detect that a synced record changed.

None of them is a Factorial Action: they keep `factorial.enabled: false`
(`fcode-factorial-actions`). The sync webhook carries no challenge header to
check: the process trusts only the ids in the body and re-reads everything
through the public API with the company's installation token.

`run(parameters)` (from `record-sync`, never override) reads the run's rows,
resolves each to a Factorial record, reads the records by id with the
vertical's `fetchPage`, calls `beforeSync`, hands the items to the mode, and
reports every row. A row it can't resolve or read is reported `invalid` before
your code sees it.

Each item your hooks receive:

```javascript
{
  syncable_sync_run_id, // the run row
  syncable_id,          // the Factorial record id
  resource_syncable_type,
  external_identifier,  // Acme's id from a previous sync, or null (create vs update)
  sync_state,
  record,               // the record as the public API returns it
}
```

## The compensation record

`CompensationsSync` reads `compensations/payroll_run_employees_compensations`:
one aggregated amount per (payroll run, employee, payroll concept), with `id`,
`payroll_run_id`, `payroll_run_employee_id` (the **employee id**, despite the
name), `payroll_concept_id`, `result_type`, `amount_strategy_type`, `amount`
(minor units), `created_at`, `updated_at`.

It has **no payroll codes, unit or period**. Resolve what Acme needs with the
inherited helpers (lazy and memoized per run):

| Acme needs | Resolve with |
|---|---|
| Employee code | `await this.employeeCodes()` → `Map(employeeId → code)` |
| Legal entity | `await this.employeeLegalEntityIds(ids)` → `Map(employeeId → legalEntityId)` |
| Legal entity code | `(await this.installationSettings()).legalEntityCodeById` |
| Payroll concept code | `resolvePayrollConceptCode((await this.installationSettings()).payrollConceptCodes, legalEntityId, conceptId)` |
| Unit (`money` / `time` / `unit` / `distance`) | `readConceptUnitTypes(this.factorialClient, conceptIds)` → `Map(conceptId → unit)` |
| Period end (`effective_on`) | `this.factorialClient.compensations.payrollRuns.all({ query: { "ids[]": … } })`, its `ends_on` |

`resolvePayrollConceptCode` and `readConceptUnitTypes` are exported next to the
class: `const { readConceptUnitTypes, resolvePayrollConceptCode } =
fcode.import("compensations-sync")`.

`mapRecord` is **synchronous**, so load every lookup once in `beforeSync`,
which runs before the mode and is awaited:

```javascript
// modules/sync/sync.js — shared by both modes
const CompensationsSync = fcode.import("compensations-sync");
const { readConceptUnitTypes, resolvePayrollConceptCode } = CompensationsSync;
const RecordSync = fcode.import("record-sync");

async function loadLookups(sync, items) {
  const employeeIds = items.map((i) => i.record.payroll_run_employee_id);
  const settings = await sync.installationSettings();
  return {
    employeeCodes: await sync.employeeCodes(),
    legalEntities: await sync.employeeLegalEntityIds(employeeIds),
    units: await readConceptUnitTypes(
      sync.factorialClient,
      items.map((i) => i.record.payroll_concept_id)
    ),
    conceptCodes: settings.payrollConceptCodes,
  };
}

function toAcmeRecord(record, lookups) {
  const employeeId = String(record.payroll_run_employee_id);
  const employeeCode = lookups.employeeCodes.get(employeeId);
  if (!employeeCode) throw RecordSync.invalid(`Employee ${employeeId} has no Acme code`);

  const legalEntityId = lookups.legalEntities.get(employeeId);
  const conceptCode = resolvePayrollConceptCode(
    lookups.conceptCodes, legalEntityId, record.payroll_concept_id
  );
  if (!conceptCode) {
    throw RecordSync.invalid(`Concept ${record.payroll_concept_id} has no Acme code`);
  }

  const unit = lookups.units.get(String(record.payroll_concept_id));
  return { employeeCode, conceptCode, unit, amount: toAcmeAmount(record.amount, unit) };
}
```

`toAcmeAmount` and `toCsv` stand for the vendor-specific parts: `amount` is in
minor units (cents for `money`), and the vendor decides how a `time` amount is
written (hours, minutes). Modes A and B below are alternatives: keep one
`AcmeSync` class in `modules/sync/sync.js`, next to these helpers.

## Mode A — file export (`FileSync`)

Map each item to rows (`[]` skips it as a no-op success), render all rows into
one file. The mode uploads it as a sync run output and reports the statuses; no
`afterSync`, no manual upload.

```javascript
const FileSync = fcode.import("file-sync");

class AcmeSync extends FileSync(CompensationsSync) {
  get externalSystem() { return "acme"; }

  async beforeSync(items) {
    this.lookups = await loadLookups(this, items);
  }

  mapRecord({ record }) {
    return [toAcmeRecord(record, this.lookups)];
  }

  render(rows) {
    return { content: toCsv(rows), prefix: "acme-compensations" }; // extension/contentType default to csv
  }
}

module.exports = AcmeSync; // the class itself, not { AcmeSync }
```

If the upload fails, every item that contributed rows is reported `failed`.

## Mode B — API push (`ApiSync`)

Deliver each item as it's processed. Use `external_identifier` to decide
between create and update. `afterSync(items, results)` is an optional
post-loop flush; a throw there fails the rows that had succeeded.

```javascript
// modules/acme-client/acme-client.js
function createAcmeClient() {
  const baseUrl = process.env.ACME__API_BASE_URL;
  const apiKey = process.env.ACME__API_KEY;
  if (!baseUrl) throw new Error("Missing ACME__API_BASE_URL team variable.");
  if (!apiKey) throw new Error("Missing ACME__API_KEY team variable.");

  async function send(method, path, body) {
    const response = await fetch(`${baseUrl.replace(/\/+$/, "")}${path}`, {
      method,
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey}` },
      body: JSON.stringify(body),
    });
    if (response.ok) {
      const text = await response.text();
      return text ? JSON.parse(text) : {};
    }
    const err = new Error(`Acme API ${method} ${path} failed (status ${response.status})`);
    if (response.status === 400 || response.status === 422) err.syncStatus = "invalid";
    throw err;
  }

  return {
    createCompensation: (r) => send("POST", "/compensations", r),
    updateCompensation: (id, r) => send("PUT", `/compensations/${id}`, r),
  };
}

module.exports = { createAcmeClient };
```

```javascript
// modules/sync/sync.js
const ApiSync = fcode.import("api-sync");
const { createAcmeClient } = fcode.import("acme-client");

class AcmeSync extends ApiSync(CompensationsSync) {
  get externalSystem() { return "acme"; }

  async beforeSync(items) {
    this.acme = createAcmeClient(); // fail fast on missing credentials
    this.lookups = await loadLookups(this, items);
  }

  async process({ record, external_identifier }) {
    const body = toAcmeRecord(record, this.lookups);
    if (external_identifier) await this.acme.updateCompensation(external_identifier, body);
    else await this.acme.createCompensation(body);
  }
}

module.exports = AcmeSync;
```

A throwing `beforeSync` fails every item of the run, which is right for missing
credentials.

**Don't name a module after a base module.** Module slugs resolve up the
workspace hierarchy, so a module of your own called `sync-api`, `api-sync`,
`file-sync` or `record-sync` shadows the base's and breaks the framework. Only
`sync` is meant to be yours.

## Optional: validate before the export

Factorial calls the inherited `validate` process before it creates a run, and
shows the user what would block the export. Opt in by overriding
`validate(item)` on the same class; it's never called during a run. Return
nothing when the record is fine, or error keys:

```javascript
async validate({ record }) {
  const codes = await this.employeeCodes();
  if (!codes.get(String(record.payroll_run_employee_id))) {
    return [this.missingField("employee_code")]; // → [{ key: "missing_employee_code" }]
  }
  return [];
}
```

## Status semantics

`run()` reports one status per item:

| Status | Meaning | How to signal |
|---|---|---|
| `success` | Delivered, or intentionally skipped | `process()` returns normally; `mapRecord()` returns rows, or `[]` to skip |
| `invalid` | Data or configuration problem — retrying won't help | throw `RecordSync.invalid(message)`, or an error with `syncStatus = "invalid"` |
| `failed` | Transient error (network, 5xx, upload failure) — retryable | throw anything else |

## Required Team Variables

| Variable | Used by | Notes |
|---|---|---|
| `FACTORIAL_TOKEN` | both modes | The company's installation token for the public API; auto-populated in the cloud |
| `FACTORIAL_BASE_URL` | both | Optional API base URL override |
| `ACME__API_BASE_URL` | API mode | Vendor API base URL |
| `ACME__API_KEY` | API mode | Vendor bearer token (sensitive) |

Vendor variables follow the `VENDOR__KEY` convention (matches
`this.variableName("API_KEY")`). How to obtain `FACTORIAL_TOKEN` locally is in
`fcode-cli`.

The run needs the API scope of its vertical (`compensations` here) on the
integration's token, and Factorial must have registered the integration before
the first run (`fcode-ama`).

## Adapting to a real vendor — checklist

1. Pick the vertical and the mode; compose them in `modules/sync`.
2. Rename `externalSystem`, the class, the file prefix and the `ACME__*`
   variables to the vendor.
3. Map the record: resolve the codes the vendor needs in `beforeSync`, and
   throw `RecordSync.invalid` when one is missing.
4. File mode: rewrite `render` for the vendor's format (columns, separator,
   header — real vendor formats often omit the header row).
5. API mode: rewrite the client's endpoints, auth scheme and create/update
   split.
6. Add `validate` if the vendor rejects records Factorial could catch first.
7. Keep the status semantics: data problems → `invalid`, transient errors →
   `failed`, skips → `success`.
