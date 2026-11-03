# Reference: inbound import ("Acme Payroll results")

An App that **imports data into Factorial** (external system → Factorial) and
tracks every import as an integrations-framework **sync run**: one run per
import, one **row** per imported record, so the customer sees in Factorial which
records were imported and which failed, and why. The example imports payroll
results from an invented vendor, **Acme Payroll**, into a Factorial payroll run.

It is the inbound counterpart of the outbound framework: same base workspace
(`base-integration-app`), same conventions (plain snake_case dicts, a
`run(parameters)` entry point, hooks the App implements, a `results` list back).

API data model: `https://apidoc.factorialhr.com/docs/inbound-integrations`.
Needs Factorial API version `2026-10-01` or later (the modules' default).

## Architecture

```
App process (form submit, schedule, webhook)
      │  new AcmePayrollImport().run(parameters)
      ▼
modules/payroll-import  ──extends──▶  inbound-record-sync (InboundRecordSync)
      │ fetchItems / process                │ fetch → open → declare → process → report → finish
      ▼                                     ▼
Acme API (read)                       inbound-sync (InboundSyncClient)
Factorial API (write the records)           │ sync_runs, syncable_sync_runs/bulk_upsert
                                            ▼
                                      Factorial (per-row status: success / invalid / failed)
```

Two inherited modules, both from `base-integration-app-<lang>`:

| Module | Export | Use it for |
|---|---|---|
| `inbound-record-sync` | `InboundRecordSync` base class | **The default.** Extend it; implement `fetchItems` and `process` (or `processBatch`) |
| `inbound-sync` | `InboundSyncClient` | The calls underneath: open/declare/report/finish by hand, abort stale runs, read a run back. Use directly only when the base class doesn't fit (see Example C and Stale runs) |

Workspace layout:

```
processes/payroll-import/   # entry point: form, schedule or webhook
modules/payroll-import/     # the InboundRecordSync subclass
modules/acme-client/        # thin vendor HTTP client
```

## Setup

- **Inherit the base.** An inbound import needs the **Integrations-framework
  app** type: it makes the App inherit `base-integration-app-<lang>` when it is
  created. Until the base's `stable` alias includes the inbound modules, pin
  the version that ships them in `settings.json` `parentTeams`:

  ```json
  "parentTeams": ["base-app-js", { "slug": "base-integration-app-js", "version": "2.5.0" }]
  ```

- **Credentials** come from the inherited `factorial-sdk` module: the
  installation's `FACTORIAL_TOKEN` (renewed before it expires) or
  `FACTORIAL_API_KEY`, and `FACTORIAL_BASE_URL` (defaults to
  `https://api.factorialhr.com`). Nothing to wire.
- **The integration must be declared in Factorial with the inbound capability,
  and installed by the company.** Otherwise opening a run is refused ("integration
  not installed") and no run opens. Factorial infers company and integration from
  the token; `company_id` / `integration_uuid` in `parameters` are only a
  fallback (a uuid that isn't the token's own is refused with a 403).

## The `InboundRecordSync` contract (inherited, never reimplement)

`run(parameters)` is the entry point. **Do not override it.** In order:

1. **`fetchItems(parameters)`** — the App reads the external system and returns
   the items: objects with an `external_identifier` (the row's key in the run),
   an optional `syncable_type` and anything else `process` needs. **Called
   before the run opens**: an empty list returns `[]` and opens no run, and a
   throw leaves no run behind.
2. Opens the run (`syncableType`, `syncParameters(parameters)`).
3. Declares **every** row as `running` before any outcome is reported.
4. `beforeSync(items, parameters)` — optional prep; a throw fails every row.
5. Per batch of `processBatchSize` items, **`processBatch(entries, parameters)`**,
   whose default calls **`process(item, parameters)`** once per item.
6. Reports each result **as soon as it settles** (every `batchSize` results, on
   `this.flush()`, and after each batch), so the customer sees progress.
7. `afterSync(items, results, parameters)` — optional; a throw is logged. A
   result settled again here is reported before the run closes.
8. Finishes the run: declared rows never reported fail as `not_reported`. On an
   unexpected error it **cancels** the run (`run_cancelled`) and rethrows.

It returns `results`: one object per item, `{ external_identifier,
syncable_type, status, syncable_id?, error_messages?, reported }`.

What the subclass declares:

| Member | Required | Meaning |
|---|---|---|
| `externalSystem`, `dataType` | yes | Slugs: log prefix, `datastoreKey(key)`, `variableName(key)` (same helpers as outbound) |
| `syncableType` | yes | Type of the rows, e.g. `compensations/compensation` (see Gotchas) |
| `fetchItems(parameters)` | yes | The items to import |
| `process(item, parameters)` | one of the two | Write one item; return the Factorial id it wrote (optional) |
| `processBatch(entries, parameters)` | one of the two | Write several items at once; settle each result yourself |
| `syncParameters(parameters)` | no | The run's sync parameters, e.g. `{ payroll_run_id: 7 }` (links the run to the payroll run it imports into) |
| `processBatchSize` | no | Items per `processBatch` call; default `batchSize` (200); `null` = all in one call |
| `beforeSync`, `afterSync` | no | Prep / wrap-up |

Constructor options: `{ timeoutMs, batchSize, logger, apiVersion }` (Python:
`timeout`, `batch_size`, `logger`, `api_version`).

## Entry process — `processes/payroll-import/index.js`

```javascript
const AcmePayrollImport = fcode.import("payroll-import");

async function main() {
  // { payroll_run_id, accounting_month } from the form or the schedule
  return new AcmePayrollImport().run(fcode.context.parameters);
}

module.exports = { main };
```

## Example A — one Factorial write per item (`process`)

The Acme client and the Factorial endpoint below are illustrative: use the
vendor's real API and the Factorial resource the import writes to.

```javascript
const InboundRecordSync = fcode.import("inbound-record-sync");
const { createFactorialClient } = fcode.import("factorial-sdk");
const acme = fcode.import("acme-client");

class AcmePayrollImport extends InboundRecordSync {
  get externalSystem() { return "acme"; }
  get dataType() { return "compensations"; }
  get syncableType() { return "compensations/compensation"; }

  syncParameters(parameters) {
    return { payroll_run_id: parameters.payroll_run_id };
  }

  async beforeSync(items, parameters) {
    this.factorial = createFactorialClient();
    this.conceptByWageType = await acme.wageTypeMapping();
  }

  async fetchItems(parameters) {
    const rows = await acme.payrollResults(parameters.accounting_month);
    return rows.map((row) => ({
      external_identifier: `${row.personnelNumber}:${parameters.accounting_month}`,
      row,
    }));
  }

  async process(item, parameters) {
    const conceptId = this.conceptByWageType[item.row.wageType];
    if (!conceptId) throw InboundRecordSync.invalid(`Wage type ${item.row.wageType} is not mapped`);
    const { data } = await this.factorial.compensations.payrollResults.create({
      body: { payroll_run_id: parameters.payroll_run_id, concept_id: conceptId, amount: item.row.amount },
    });
    return data.id; // reported as the row's syncable_id
  }
}

module.exports = AcmePayrollImport;
```

## Example B — one bulk write for many items (`processBatch`)

When the Factorial endpoint writes many records at once (or must be atomic),
override `processBatch` and settle each result with the helpers. Settled results
are queued and sent in `bulk_upsert` batches; call `this.flush()` to report what
is settled **now** (for example right after a slow external call fails a
client's rows), so the customer's view updates before the batch ends.

```javascript
class AcmePayrollImport extends InboundRecordSync {
  // … externalSystem / dataType / syncableType / fetchItems as in Example A …

  get processBatchSize() { return null; } // every item in one call: one atomic write

  async processBatch(entries, parameters) {
    const writable = [];
    for (const { item, result } of entries) {
      const conceptId = this.conceptByWageType[item.row.wageType];
      if (!conceptId) await this.settle(result, `Wage type ${item.row.wageType} is not mapped`);
      else writable.push({ result, body: toPayrollResult(item, conceptId, parameters) });
    }
    await this.flush(); // the invalid rows show up now

    const { data, error } = await this.factorial.compensations.payrollResults.bulkCreate({
      body: { payroll_results: writable.map((w) => w.body) },
    });
    for (const [index, { result }] of writable.entries()) {
      if (error) await this.fail(result, "Factorial refused the payroll results");
      else await this.succeed(result, data[index].id);
    }
  }
}
```

| Helper | Sets | `error_messages` key |
|---|---|---|
| `succeed(result, syncableId?)` | `success` | — (a `syncableId` that isn't an integer or digit string fails the row instead) |
| `settle(result, message)` | `invalid` | `sync_validation_error` |
| `fail(result, message)` | `failed` | `sync_api_error` |

A result still `pending` after `processBatch` returns is failed as
`not processed`; if `processBatch` throws, only that batch's pending rows fail
and the next batch still runs.

## Example C — the run opened in one process, imported in another

A form must answer within its request timeout, and should tell the user right
away when an import is already running. So the form **opens and declares** the
run with the client, then hands the run to an async worker, which extends
`InboundRecordSync` as usual and receives the run as `parameters.sync_run`. The
run is a plain JSON object, so it travels in the worker's parameters as is.

```javascript
// processes/payroll-import-launch/index.js — the form's submit
const { InboundSyncClient, SyncRunAlreadyRunningError, InboundSyncApiError } = fcode.import("inbound-sync");

async function main() {
  const { payroll_run_id, accounting_month, personnel_numbers } = fcode.context.parameters;
  const client = new InboundSyncClient();
  let run;
  try {
    run = await client.startRun("compensations/compensation", { parameters: { payroll_run_id } });
  } catch (error) {
    if (error instanceof SyncRunAlreadyRunningError || error instanceof InboundSyncApiError) {
      return { message: "An import is already running or could not start. Try again in a few minutes." };
    }
    throw error;
  }
  await client.declarePlan(run, personnel_numbers.map((n) => `${n}:${accounting_month}`));
  // A separate execution of the worker, with the run in its parameters; the form answers now.
  await fcode.processes.run("payroll-import", {
    parameters: { payroll_run_id, accounting_month, sync_run: run },
  });
  return { message: "Import started." };
}
```

With `sync_run` given, `run()` opens nothing, declares only fetched items the
run doesn't hold yet, and still finishes the run: rows declared by the launcher
that the worker never fetched fail as `not_reported`.

## Using the client directly

`InboundSyncClient` (`inbound-sync`) is what the base class calls. Items are
plain objects (`{ external_identifier, status, syncable_id?, error_messages?,
syncable_type? }`); the run is the plain object `startRun` returns, and every
write updates it in place and returns it.

| Method | Does |
|---|---|
| `startRun(syncableType, { companyId?, integrationUuid?, parameters? })` | Opens a run; returns the run object |
| `declarePlan(run, items)` | Declares rows as `running` (items or plain external identifiers) |
| `reportOutcome(run, items)` | Reports rows, each with an explicit `status` |
| `finishRun(run)` | Fails every declared-but-unreported row (`not_reported`) |
| `cancelRun(run, reason?)` | Fails every open row (`run_cancelled`, with `reason`) |
| `abortRun(runId, { reason? })` | Closes a run whose object is lost (crashed worker), from the API |
| `abortOpenRunsForIntegration(integrationUuid, { since?, reason? })` | Aborts every still-running run of the integration (default: last two hours) — e.g. on uninstall. Throws before any request when the uuid is missing or blank. Runs carry no origin: an integration that also exports would have its running outbound runs aborted too |
| `listRunsForIntegration(uuid, createdAtGteq)` | Lists the integration's runs (API field names, snake_case), e.g. to check whether one is still running. Throws when the uuid is missing or blank |

## Status semantics

| Status | When | Shown to the customer |
|---|---|---|
| `running` | Declared, not finished yet | In progress |
| `success` | Written into Factorial (`syncable_id` links the record) | Imported |
| `invalid` | Data or setup problem: retrying won't help (`InboundRecordSync.invalid(…)`) | The message, to fix the data or mapping |
| `failed` | Anything else, may work next time; also `not_reported` and `run_cancelled` | The message |

Write `error_messages` for an end user: Factorial shows them to the customer.

## Gotchas

- **Fetch before you open.** Factorial never closes a run with no rows and
  blocks the next run for about an hour. `run()` handles this; when using the
  client, open a run only once you know what you will import.
- **Declare the whole plan before any outcome.** Factorial closes a run after
  any write that leaves no row `running`. `run()` does this; by hand, call
  `declarePlan` with every row first.
- **One open run per company and integration.** On the default API version
  `2026-10-01` a second open run is refused as an `InboundSyncApiError` with
  status 422 (its message is translated: never match the text); from
  `2027-01-01` it is a 409 `SyncRunAlreadyRunningError`. What to do is the
  App's decision: tell the user, retry later, or abort the old run.
- **Only registered syncable types**: today `compensations/compensation`,
  `expenses/expense`, `finance/vendor`, `finance/financial_document` and the
  `employee_updates/…` types. An unknown type refuses the whole write.
- **Writes after the run opens never throw** — a refused batch is logged and
  counted in `run.unsent`; bad input still throws. Opening a run and the reads
  do throw.
- **Only one process finishes or cancels a run**: it closes the rows its own
  run object still sees open.
- **Logs never carry identifiers or messages** (external identifiers and
  messages may hold personal data). Keep your own logs to counts and run ids.
- **Ids above 2^53** (cell-scoped Factorial ids) are sent exactly as JSON
  numbers; pass them as strings of digits in JavaScript.

## Python variant

Same modules, snake_case: `fcode.import_module("inbound-record-sync").InboundRecordSync`,
properties `external_system` / `data_type` / `syncable_type`, hooks
`fetch_items`, `process`, `process_batch`, `sync_parameters`, `before_sync`,
`after_sync`, `process_batch_size`; helpers `succeed` / `settle` / `fail` /
`flush`; `raise self.invalid("...")`. The client is
`fcode.import_module("inbound-sync").InboundSyncClient()` with `start_run`,
`declare_plan`, `report_outcome`, `finish_run`, `cancel_run`, `abort_run`,
`abort_open_runs_for_integration`, `list_runs_for_integration`. The run dict is the same JSON document in both languages, so a
launcher in one can hand it to a worker in the other.

## Adapting to a real vendor — checklist

1. Inherit `base-integration-app-<lang>` (pinned until `stable` ships the inbound modules).
2. Confirm the integration is declared with the inbound capability and the syncable type is registered.
3. Pick the `external_identifier`: stable per record and import (e.g. `personnel_number:month`).
4. `fetchItems` reads the vendor; nothing fetched means no run.
5. `process` for one write per item, or `processBatch` (+ `flush`) for bulk/atomic writes.
6. Raise `invalid(...)` for data/mapping problems, with a message the customer can act on.
7. A form that must answer fast: open and declare with `InboundSyncClient`, hand the run to an async worker as `sync_run` (Example C).
8. On uninstall, `abortOpenRunsForIntegration` so no run stays open. Store the
   run's `integration_uuid` when you open it: the call needs it.
