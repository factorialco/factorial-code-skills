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
per imported record. Factorial shows the customer which items succeeded and
which failed. The helper wraps Factorial's inbound integrations API and is on
`fcode` in every process: `fcode.inboundSync` (JavaScript),
`fcode.inbound_sync` (Python). Nothing to inherit or configure — it is built
from the installation's `FACTORIAL_TOKEN` / `FACTORIAL_BASE_URL` (see
`fcode-cli` for those variables) and refreshes an expired token itself.

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
- **Start a run only once you know what you will import.** A run that never
  gets an item can't be closed by anyone — `finishRun` / `cancelRun` on an empty
  handle send nothing and log a warning — and it blocks the next `startRun`
  until Factorial's reaper closes it, about an hour later.
- **One open run per company and integration.** A second `startRun` raises
  `SyncRunAlreadyRunningError` (HTTP 409). What to do is **the App's decision**
  — tell the user an import is already running, queue it, or abort the old run;
  the helper doesn't guess.
- **Only one process finishes or cancels a run.** They close the rows *the
  local handle* still sees open, so with several workers sharing copies of a
  handle, one finishing fails items another is still processing.
- **Writes after `startRun` never throw.** Declaring, reporting, finishing and
  cancelling are best effort: a failed batch is logged and counted in
  `handle.unsent` instead of breaking the import. Check `handle.unsent` at the
  end if the user should hear about it. Bad input (an empty plan, a missing or
  unknown status) *does* throw, before anything is sent.
- **Every reported item needs an explicit `status`.** `syncableId` is sent on
  `success` only; `errorMessages` on `failed` / `invalid` only — and Factorial
  shows those messages to the customer, so write them for an end user.
- **Use the helper on `fcode`, don't build `FcodeInboundSync` yourself** inside
  a process: the runtime one already has the installation's credentials and the
  token refresh wired in.
- **No local-only mode.** Without `FACTORIAL_TOKEN` and `FACTORIAL_BASE_URL`,
  `startRun` raises `InboundSyncError` — see Running locally.
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

`startRun` accepts an optional `companyId` and `integrationUuid` (the App's
marketplace integration uuid), and `abortOpenRunsForIntegration` /
`listRunsForIntegration` require the uuid. **The App owns that value**: it's
fixed per integration, so keep it in a team variable (for example
`INTEGRATION_UUID`, read through `fcode.env`) or as a constant in the code —
never hardcode it in several processes.

## The run lifecycle

1. **Start the run** — opens an empty run and returns a **handle** you pass to
   every later call. The argument is the **syncable type**, the kind of record
   the run imports (for example `compensations/compensation`).
2. **Declare the plan** — every item you are about to import, all as
   `running`, right after starting. Items are external identifiers (strings) or
   item objects.
3. **Report outcomes** — each item's final status once your import is done
   with it, as many calls as you like (for example one per batch). **Factorial
   closes the run on its own once no item is `running`.**
4. **Finish or cancel** — only when the flow breaks:
   - **finish**: every declared-but-unreported item fails as `not_reported`;
   - **cancel**: every open item fails with `run_cancelled` and your reason
     (blank → default; over 200 characters → cut and suffixed ` […]`).

A run may mix syncable types: an item's own `syncableType` overrides the
handle's, and rows are keyed by (syncable type, external identifier).
Declaring a reported item again, or reporting it `running`, reopens it.

```javascript
// JavaScript
const handle = await fcode.inboundSync.startRun("compensations/compensation", {
  integrationUuid: fcode.env.INTEGRATION_UUID,
});

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

handle = fcode.inbound_sync.start_run(
    "compensations/compensation",
    integration_uuid=fcode.env.INTEGRATION_UUID,
)

fcode.inbound_sync.declare_plan(handle, ["emp-1", "emp-2", "emp-3"])

# ... import each record into Factorial and collect its result ...

fcode.inbound_sync.report_outcome(handle, [
    SyncableItem("emp-1", "success", syncable_id=4821),
    SyncableItem("emp-2", "invalid", error_messages={"base": "Missing IBAN"}),
])

# emp-3 was declared but never reported: finishing fails it as "not_reported"
fcode.inbound_sync.finish_run(handle)
```

Python items may also be `dict`s or plain identifier strings; JavaScript items
are `{ externalIdentifier, status?, syncableId?, errorMessages?, syncableType? }`.

## Item statuses

| Status | Meaning |
|---|---|
| `running` | Declared; your import isn't done with it yet |
| `success` | Imported; may carry `syncableId`, the id of the Factorial record created |
| `failed` | Your import tried and couldn't, for example a Factorial write failed |
| `invalid` | Its data can't be imported as is, for example a missing mapping — fixing the data fixes it |

A run's own status is `running`, `succeeded`, `succededwitherrors` (sic) or
`errored`.

## Handing a run to a worker process

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

Declare in the launcher, not the worker: the plan must exist before the first
outcome, whichever process reports it. If several workers report, only one of
them (or a final step) finishes the run.

## Aborting stale runs on uninstall

A run left open blocks the next one until Factorial closes it, an hour after it
opened. The App's uninstall process (the `uninstall` lifecycle slug — see
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
| `startRun` / `start_run` | `SyncRunAlreadyRunningError` (409, a run is open); `InboundSyncError` (no token or base URL, network failure); `InboundSyncApiError` (any other refusal: token still rejected after refresh, integration not installed, 5xx); a plain error for an empty type | — |
| `declarePlan` / `declare_plan` | Empty list | Any HTTP or network failure |
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
**Opening a run is never retried** — a retry after a timeout could be refused
by the run the first attempt opened — so a failed `startRun` is the user's to
retry.

## Reading a run back

For debugging and tooling: `getRun(runId)` (one run),
`listRunsForIntegration(integrationUuid, createdAtGteq)` (runs created since a
date, every page), `readResults(runId)` (the run's items). Python:
`get_run`, `list_runs_for_integration`, `read_results`. A status the helper
doesn't know is returned as received and logged.

## Running locally

The helper always talks to a real Factorial API. For `fcode run`, put
`FACTORIAL_TOKEN` and `FACTORIAL_BASE_URL` in `variables.local.env` (how to get
the token: `fcode-cli` §Getting secret values for local runs), pointing at a
Factorial environment you control or at a stub of the endpoints you need.
Without them, `startRun` raises.
