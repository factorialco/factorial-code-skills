---
name: fcode-javascript
description: Write JavaScript (Node.js v22) for Factorial Code processes and modules — the async main() entry point with module.exports = { main }, fcode.context.parameters, fcode.import(), datastore/storage/env helpers, auto-installed dependencies, and return-value formats. Use when creating or editing .js process or module code for Factorial Code (fcode).
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — JavaScript

Guidelines for writing JavaScript that runs on Factorial Code. Runtime is
**Node.js v22**. For the platform model (processes, modules, datastore) see
`fcode-core-concepts`.

| Aspect | Guideline |
|---|---|
| Runtime | Node.js v22 |
| Process entry file | `index.js` |
| Entry point | `async function main()` |
| Export (required) | `module.exports = { main }` |
| Parameters | `fcode.context.parameters` |
| Variables | `process.env.X` or `fcode.env.X` |
| Import a module | `fcode.import("module-slug")` |

## Gotchas

- **Always export** `main`: `module.exports = { main }`. Without it the process
  won't run.
- **Never call `main()` yourself** — Factorial Code invokes it.
- **`fcode.import()` names must be hardcoded string literals**, never variables:
  `fcode.import("shopify-client")` ✅, `fcode.import(name)` ❌.
- **Datastore stores only strings/numbers** — `JSON.stringify` objects before
  `set`, parse after `get`.
- Use `async/await` for all async work; wrap the main flow in `try/catch` and
  throw actionable errors. Use `const`/`let`, never `var`.
- Never hardcode or log secrets — read them from `process.env`.

## Process template

```javascript
async function main() {
  const { parameters } = fcode.context;

  // Your code here

  return { message: "Success!" };
}

module.exports = { main };
```

## Helpers

```javascript
// Execution / process / schedule metadata
const { id, comment } = fcode.execution;
const { id: processId, name: processName } = fcode.execution.process;
const { id: scheduleId } = fcode.execution.schedule; // when run from a schedule
const timezone = fcode.execution.timezone;

// Environment variables (secrets/config)
const apiKey = process.env.API_KEY; // or fcode.env.API_KEY

// Import a Factorial Code module (hardcoded name only)
const { myFunc } = fcode.import("module-name");
const { myFunc: v1 } = fcode.import("module-name", "v1.0"); // pinned version

// Run another process
await fcode.processes.run("process-identifier", options);
```

## Logging

`console.log/debug/info/warn/error` — each maps to the matching log level.
Never log secrets.

## Dependencies

External npm packages install automatically — just `require` them. When the
import name differs from the package name, declare it with `@add-package`:

```javascript
// @add-package axios
const axios = require("axios");
```

## Datastore & storage

```javascript
// Datastore (strings/numbers only)
await fcode.datastore.set("key", "value");
await fcode.datastore.set("key", JSON.stringify({ name: "John", age: 30 }));
const value = await fcode.datastore.get("key");
await fcode.datastore.del("key");

// Storage (files)
const fs = require("node:fs");
const path = require("node:path");
const localPath = path.join(process.env.TMP_DATA_DIR, "localfile.txt");

await fcode.storage.upload("path/myfile.txt", fs.createReadStream(localPath));
const files = await fcode.storage.list();
const stream = await fcode.storage.download("path/myfile.txt");
stream.pipe(fs.createWriteStream(localPath));
await fcode.storage.delete("path/myfile.txt");
```

**Local disk:** write temp files under `process.env.TMP_DATA_DIR`.

**Form file uploads:** a form file field (`"ui:widget": "file"`) is uploaded to
Storage before execution and arrives as an `fcode.storage://…` reference (an
array if multiple files). Strip the prefix to download:

```javascript
const { context: { parameters } } = fcode;
const stream = await fcode.storage.download(
  parameters.inputFile.replace("fcode.storage://", "")
);
```

## Return values

```javascript
// Standard
return { message: "Success!" };

// Custom HTTP status (webhooks)
return { status: 404, body: { message: "Not found" }, headers: { "Content-Type": "application/json" } };

// Transient (not persisted in execution results)
return { transient: true, data: sensitiveData };
```
