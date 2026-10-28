---
name: fcode-inbound-sync
description: Track an import into Factorial as an inbound sync run with the fcode.inboundSync / fcode.inbound_sync helper — start a run, declare the plan, report each item's outcome, finish or cancel, hand a run to a worker process, abort stale runs on uninstall, and what raises versus what is best effort. Use when a Factorial Code (fcode) process brings data from an external system into Factorial and the per-item result should show up in Factorial.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Inbound sync tracking

An App that **imports data into Factorial** (an external system → Factorial)
reports each import as a **sync run**: one run per import, one **item** (row)
per imported record, so Factorial can show the customer which items succeeded
and which failed. The helper wraps Factorial's inbound integrations API and is on
`fcode` in every process: `fcode.inboundSync` (JavaScript),
`fcode.inbound_sync` (Python). Nothing to inherit — it is built from the
installation's `FACTORIAL_TOKEN` / `FACTORIAL_BASE_URL` (see `fcode-cli` for
those variables) and refreshes an expired token itself. `FACTORIAL_BASE_URL` may
be missing in production — see Gotchas.

API data model: `https://apidoc.factorialhr.com/docs/inbound-integrations`.
Platform docs: `/docs/building-apps/inbound-sync/` on
`https://code.factorialhr.com`. Needs Factorial API version `2026-10-01` or
later (the helper's default).

## Gotchas

- **Declare the plan before reporting any outcome.** Factorial closes a run
  after any write that leaves no item `running`, and one report call may go out
  as several requests (per syncable type, in batches of 200). Items reported
  before the rest are declared can close the run early, and later writes land
  on a finished run.
- **Start a run only once you know what you will import.** Factorial never
  closes a run that gets no item, and neither can the App — `finishRun` /
  `cancelRun` on an empty handle send nothing and log a warning. It stays open
  and blocks the next `startRun` until an hour after it started.
- **One open run per company and integration** — sync parameters don't change
  that scope. What to do on a second `startRun` is **the App's decision** (tell
  the user an import is already running, queue it, or abort the old run); the
  helper doesn't guess. **How it surfaces depends on the API version:**
  - on `2026-10-01` — the helper's default, and the only version the runtime
    `fcode` helper uses — it is an `InboundSyncApiError` with `status` 422, the
    same status as other refusals (integration not installed, missing company).
    Its message is translated into the company's language, so never match the
    text;
  - from `2027-01-01` Factorial answers 409 and the helper raises
    `SyncRunAlreadyRunningError`. Getting it today means building your own
    `FcodeInboundSync` with that `apiVersion` / `api_version` — a beta
    version; see "Use the helper on `fcode`" below.
- **Use only syncable types Factorial has registered.** The syncable type
  (`compensations/compensation`, …) must be one of the public types the
  monolith's syncable type registry knows; an unknown one is refused. A new
  kind of record needs its type added in the Factorial monolith first (a value
  in the public `SyncableItem` type enum plus a registry entry from the owning
  component) — check the monolith before relying on a new type.
- **Only one process finishes or cancels a run.** They close the rows *the
  local handle* still sees open, so with several workers sharing copies of a
  handle, one finishing fails items another is still processing.
- **Writes after `startRun` never throw** — a failed batch is counted in
  `handle.unsent` instead of breaking the import. Check it at the end if the
  user should hear about it. Bad input still throws; see What raises and what
  is best effort.
- **Every reported item needs an explicit `status`.** `syncableId` is sent on
  `success` only; `errorMessages` on `failed` / `invalid` only — and Factorial
  should show those messages to the customer, so write them for an end user.
- **`FACTORIAL_BASE_URL` may be missing in production.** Factorial Code sets it
  on the `deploy-` team only for development installations; without it
  `startRun` raises a "not configured" `InboundSyncError`. Define it in the
  App's own variables (for example
  `FACTORIAL_BASE_URL=https://api.factorialhr.com/` in
  `deploy/prod/variables.env`), or build the helper with the host as below.
- **Use the helper on `fcode`, don't build `FcodeInboundSync` yourself** inside
  a process: the runtime one already has the installation's credentials and the
  token refresh wired in. The exceptions are a different API version, which the
  runtime helper can't take, and a missing `FACTORIAL_BASE_URL` (pass
  `baseUrl: "https://api.factorialhr.com"` / `base_url=...` instead of the
  variable). Then wire the same credentials in yourself:
  `new FcodeInboundSync({ token: fcode.env.FACTORIAL_TOKEN, baseUrl:
  fcode.env.FACTORIAL_BASE_URL, refreshToken: async () => (await
  fcode.refreshFactorialToken()).token, apiVersion })` — Python:
  `FcodeInboundSync(token=..., base_url=..., refresh_token=lambda:
  fcode.refresh_factorial_token()["token"], api_version=...)`.
- **Not tied to the "Integrations-framework app" type.** That type is for
  outbound connectors Factorial calls; an inbound App is a **Standard app** (see
  `fcode-ama` `references/journey.md` §2).
- The helper never logs identifiers, error messages, bodies or tokens — keep
  your own logs to the same rule (counts and run ids only).

## When to use it

Use it when the App writes records into Factorial on behalf of an external
system and the customer should see the result in Factorial — a payroll results
import, an employee import, a scheduled poll that creates records.

Don't use it when the App sends data **from** Factorial to an external system
(that's the integrations framework — `/docs/building-apps/integrations-framework/`)
or doesn't write into Factorial at all (reports, notifications, read-only
automations).

## The integration uuid

> **Note for the developer — pass this on, don't act on it.** The integration
> must be declared in Factorial with the inbound capability, and the company
> must have installed the App. Otherwise `startRun` fails with an "integration
> not installed" error, or the run opens as `errored`.

`startRun` doesn't need the integration: Factorial infers the company and the
marketplace integration from the installation's OAuth token (a uuid that
doesn't match the token's is refused). `companyId` / `integrationUuid` are a
**fallback** for when that inference fails, for example a credential that spans
several companies.

`abortOpenRunsForIntegration` and `listRunsForIntegration` do take the uuid as
an argument. **Suggestion:** take it from a run you opened —
`handle.integrationUuid` / `handle.integration_uuid`, as Factorial returned it —
and store it (for example in the datastore). Runs open under the integration
that owns the OAuth credential, which can differ from an integration uuid the
App knows. A constant or team variable (for example `INTEGRATION_UUID`, read
through `fcode.env`) also works, kept in one place, as long as it is the uuid
the runs actually use.

## The run lifecycle

1. **Start the run** — opens an empty run and returns a **handle** you pass to
   every later call. The argument is the **syncable type**, the kind of record
   the run imports (for example `compensations/compensation`). Optionally pass
   the run's **sync parameters** (see below).
2. **Declare the plan** — every item you are about to import, all as
   `running`, right after starting (see Gotchas for why first). Items are external identifiers (strings) or
   item objects.
3. **Report outcomes** — each item's final status once your import is done
   with it, as many calls as you like (for example one per batch). **Factorial
   closes the run on its own once no item is `running`.**
4. **Finish or cancel** — only when the flow breaks:
   - **finish**: every declared-but-unreported item fails as `not_reported`;
   - **cancel**: every open item fails with `run_cancelled` and your reason
     (blank → default; over 200 characters → cut and suffixed ` […]`).

**Use one syncable type per run** — the handle's. That is the usual flow and
the recommended one. Factorial does accept a run mixing types (an item's own
`syncableType` / `syncable_type` overrides the handle's, and each write goes
out as one request per type), but don't reach for it without a reason; if you
do, the plan rule above still applies to the whole run. Declaring a reported
item again, or reporting it `running`, reopens it.

```javascript
// JavaScript
const handle = await fcode.inboundSync.startRun("compensations/compensation");

await fcode.inboundSync.declarePlan(handle, ["emp-1", "emp-2", "emp-3"]);

// ... import each record into Factorial and collect its result ...

await fcode.inboundSync.reportOutcome(handle, [
  { externalIdentifier: "emp-1", status: "success", syncableId: 4821 },
  { externalIdentifier: "emp-2", status: "invalid", errorMessages: { base: "Missing IBAN" } },
]);

// emp-3 was declared but never reported: finishing fails it as "not_reported"
await fcode.inboundSync.finishRun(handle);
```

```python
# Python
from fcode_sdk import SyncableItem

handle = fcode.inbound_sync.start_run("compensations/compensation")

fcode.inbound_sync.declare_plan(handle, ["emp-1", "emp-2", "emp-3"])

# ... import each record into Factorial and collect its result ...

fcode.inbound_sync.report_outcome(handle, [
    SyncableItem("emp-1", "success", syncable_id=4821),
    SyncableItem("emp-2", "invalid", error_messages={"base": "Missing IBAN"}),
])

# emp-3 was declared but never reported: finishing fails it as "not_reported"
fcode.inbound_sync.finish_run(handle)
```

`declarePlan` / `declare_plan` takes plain identifier strings or items. Reported
items need a status: `SyncableItem`s or `dict`s in Python (a string raises
`ValueError`), `{ externalIdentifier, status, syncableId?, errorMessages?,
syncableType? }` in JavaScript.

### Sync parameters

`startRun` takes the run's sync parameters as a key/value map, stored on the
run (sent as `sync_parameters`, each value as a string):

```javascript
const handle = await fcode.inboundSync.startRun("compensations/compensation", {
  parameters: { payroll_run_id: payrollRunId },
});
```

```python
handle = fcode.inbound_sync.start_run(
    "compensations/compensation", parameters={"payroll_run_id": payroll_run_id}
)
```

- Factorial keeps every key, but only acts on the ones it knows. Today that is
  **`payroll_run_id`**: it links a payroll-results import to the payroll run it
  writes into, and when the run finishes Factorial moves that payroll run to
  review and notifies the customer. It must be the payroll run's integer id.
- The map must be a `dict` (Python) or plain object (JavaScript). Keys are
  non-blank strings (trimmed); values are non-blank strings or integers —
  floats, booleans, `null`/`None` and nested values are refused with a plain
  error before anything is sent.

## Item statuses

| Status | Meaning |
|---|---|
| `running` | Declared; your import isn't done with it yet |
| `success` | Imported; may carry `syncableId`, the id of the Factorial record created |
| `failed` | Your import tried and couldn't, for example a Factorial write failed |
| `invalid` | Its data can't be imported as is, for example a missing mapping — fixing the data fixes it |

A run's own status is `running`, `succeeded`, `succededwitherrors` (sic) or
`errored`.

## Recommended flows

### Handing a run to a worker process

The usual shape: the process behind the form (or schedule) opens the run and
declares the plan, then hands the work to a worker with
`fcode.processes.run` (see `fcode-forms` §Keep it fast, or go async).
Serialise the handle into the worker's parameters and rebuild it there. The
document is the same in both languages, so the two processes may be written in
different ones.

```javascript
// Launcher
const handle = await fcode.inboundSync.startRun("compensations/compensation");
await fcode.inboundSync.declarePlan(handle, identifiers);
await fcode.processes.run("import-worker", {
  async: true,
  parameters: { syncHandle: handle.toJSON() },
});

// Worker
const { SyncRunHandle } = require("@factorialco/fcode-sdk");
const handle = SyncRunHandle.fromJSON(fcode.context.parameters.syncHandle);
await fcode.inboundSync.reportOutcome(handle, results);
await fcode.inboundSync.finishRun(handle);
```

```python
# Launcher
handle = fcode.inbound_sync.start_run("compensations/compensation")
fcode.inbound_sync.declare_plan(handle, identifiers)
fcode.processes.run("import-worker", {
    "async": True,
    "parameters": {"sync_handle": handle.to_dict()},
})

# Worker
from fcode_sdk import SyncRunHandle

handle = SyncRunHandle.from_dict(fcode.context.parameters["sync_handle"])
fcode.inbound_sync.report_outcome(handle, results)
fcode.inbound_sync.finish_run(handle)
```

Declare in the launcher, not the worker, and let one process (or a final step)
finish the run — the two rules are in Gotchas.

### Aborting stale runs on uninstall

A run left open keeps blocking the next import (see Gotchas). The App's
uninstall process (the `uninstall` lifecycle slug — see
`fcode-factorial-actions`) closes whatever is still open, without a stored
handle:

```javascript
const aborted = await fcode.inboundSync.abortOpenRunsForIntegration(
  fcode.env.INTEGRATION_UUID,
  { reason: "App uninstalled" },
);
```

```python
aborted = fcode.inbound_sync.abort_open_runs_for_integration(
    fcode.env.INTEGRATION_UUID,
    reason="App uninstalled",
)
```

- It returns the ids of the runs it aborted and looks at runs created in the
  last two hours unless you pass `since`. A run that fails to abort is logged
  and left out; the others are still tried.
- `abortRun(runId, { reason })` / `abort_run(run_id, reason=...)` aborts one run
  whose handle is lost (a crashed worker): it rebuilds the open items from
  Factorial and fails the `running` ones, returning how many. 0 means no item
  was `running`; a run with no items at all is left open with a warning. It
  raises `InboundSyncError` when a `running` item couldn't be closed.
- To close a run you still hold, finish or cancel it instead.
- The abort needs the installation's token. If Factorial has already revoked it
  when the uninstall process runs, only the reaper closes the run.

## What raises and what is best effort

| Call | Raises | Best effort (logged, counted in `handle.unsent`) |
|---|---|---|
| `startRun` / `start_run` | `InboundSyncApiError` for any refusal; `SyncRunAlreadyRunningError` for a run already open, from API `2027-01-01` only (see Gotchas); `InboundSyncError` (no token or base URL, network failure); a plain error for an empty type or a bad sync parameter | — |
| `declarePlan` / `declare_plan` | An empty list, an item without an identifier or a value that isn't an item (never for a status) | Any HTTP or network failure |
| `reportOutcome`, `finishRun`, `cancelRun` (and snake_case) | Missing or unknown status | Same as `declarePlan` |
| `abortRun` / `abort_run` | `InboundSyncApiError` on a refused read; `InboundSyncError` when a `running` item stays open | — |
| `abortOpenRunsForIntegration` / `abort_open_runs_for_integration` | `InboundSyncApiError` when listing the runs is refused | A run that couldn't be aborted (left out of the result) |
| `getRun`, `listRunsForIntegration`, `readResults` (and snake_case) | `InboundSyncApiError` | — |

The plain error is `Error` in JavaScript and `ValueError` in Python. Errors
expose `status` (HTTP) and `key` (a recognised error key), never the response
body. Import the classes from the SDK (`require("@factorialco/fcode-sdk")` /
`from fcode_sdk import ...`) to catch them.

**Retries.** Transport errors, 429 and 5xx are retried twice (0.5 s, then 2 s),
and a 401 once after refreshing the token; each request times out after 20 s.
**Opening a run doesn't retry transport errors or 5xx** — a retry could be
refused by the run the first attempt opened — so a failed `startRun` is the
user's to retry. It still retries a 429 and refreshes a 401.

## Reading a run back

For debugging and tooling: `getRun(runId)` (one run),
`listRunsForIntegration(integrationUuid, createdAtGteq)` (runs created since a
date, every page), `readResults(runId)` (the run's items). Python:
`get_run`, `list_runs_for_integration`, `read_results`. A status the helper
doesn't know is returned as received and logged.

## Running locally

There's no local-only mode: the helper always talks to a real Factorial API,
and without a token and a host `startRun` raises `InboundSyncError`. For
`fcode run`, put
`FACTORIAL_TOKEN` and `FACTORIAL_BASE_URL` in `variables.local.env` (how to get
the token: `fcode-cli` §Getting secret values for local runs), pointing at a
Factorial environment you control or at a stub of the endpoints you need.
