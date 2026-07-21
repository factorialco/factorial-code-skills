# Reference: utility processes

Two self-contained single-process automations. Smaller than the integration
and custom-app references, but each demonstrates an end-to-end chain of
platform features worth copying.

## Time-off CSV export — Storage + signed URL + email

**Flow:** fetch time-off leaves in a date range → build a CSV → upload it to
fcode Storage → create a signed download URL → email the link with
`fcode.sendMail`.

Inputs (form): `start_date`, `end_date` (YYYY-MM-DD), `recipient_email`.

```javascript
async function main() {
  const { start_date, end_date, recipient_email } = fcode.context.parameters;
  if (!start_date || !end_date) throw new Error("start_date and end_date are required (YYYY-MM-DD)");
  if (!recipient_email) throw new Error("recipient_email is required");

  // 1. Fetch data — the SDK's .all() auto-paginates.
  const { createFactorialClient } = fcode.import("factorial-sdk");
  const factorialClient = createFactorialClient();
  const leaves = await factorialClient.timeoff.leaves.all({
    query: { from: start_date, to: end_date, include_deleted_leaves: false },
  });
  const employees = await factorialClient.employees.employees.all();
  const nameById = new Map(employees.map((e) => [e.id, e.full_name]));
  const rows = leaves.map((leave) => ({ /* enrich with nameById, flatten */ }));

  // 2. Write locally, then upload to Storage.
  const fs = require("node:fs");
  const path = require("node:path");
  const os = require("node:os");
  // TMP_DATA_DIR exists in the cloud runtime; fall back to the OS temp dir locally.
  const tmpDir = process.env.TMP_DATA_DIR || os.tmpdir();
  const filename = `timeoffs-${start_date}_${end_date}-${fcode.execution.id}.csv`;
  const localPath = path.join(tmpDir, filename);
  fs.writeFileSync(localPath, buildCsv(rows));
  const storagePath = `timeoff-exports/${filename}`;
  await fcode.storage.upload(storagePath, fs.createReadStream(localPath));

  // 3. Signed download URL — { url, expiresAt }. In the cloud it's a real
  //    signed HTTPS link; locally (`fcode run`) it's a file:// URL. Same shape,
  //    so no special-casing.
  const signed = await fcode.storage.createSignedUrl(storagePath);

  // 4. Email the link. brandedHtml (mail-helper, base-app) gives a styled body.
  const { brandedHtml } = fcode.import("mail-helper");
  await fcode.sendMail({
    to: recipient_email,
    subject: `Time-off export ${start_date} → ${end_date} (${rows.length} leaves)`,
    text: `Download (link expires): ${signed.url}`,
    html: brandedHtml({
      heading: "Your time-off export is ready",
      intro: `${rows.length} leave(s) between ${start_date} and ${end_date}.`,
      button: { label: "Download CSV", url: signed.url },
    }),
  });

  return { success: true, leaves: rows.length, storagePath, downloadUrl: signed.url };
}
```

CSV building: quote every cell and escape embedded quotes —
`` `"${String(v).replace(/"/g, '""')}"` `` — don't naïvely `join(",")` raw
values.

**Patterns:** SDK `.all()` auto-pagination · enrichment via a prefetched
`Map` (no N+1 lookups) · `TMP_DATA_DIR` with local fallback ·
`storage.upload` + `storage.createSignedUrl` · `fcode.sendMail` +
`brandedHtml`.

## XML employee documents — form file upload + XML transform + document upload

**Flow:** read an XML file uploaded through the form → find every node with
an `<Email>` element → match nodes to Factorial employees by email → inject
the selected employee fields into each node → upload the enriched XML to one
employee's Factorial documents.

Inputs (form): `xml_file` (file field), `upload_to_email`, `fields`
(multi-select, `"all"` shortcut).

```javascript
// @add-package fast-xml-parser
const { XMLParser, XMLBuilder } = require("fast-xml-parser");

/** Read a storage download stream into a string. */
const streamToString = async (stream) => {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
  }
  return Buffer.concat(chunks).toString("utf8");
};

async function main() {
  try {
    const { xml_file, upload_to_email, fields = ["all"] } = fcode.context.parameters;

    // 1. Form file fields are auto-uploaded to Storage before execution.
    //    In the cloud the parameter is an "fcode.storage://" URI; locally a plain
    //    path. Strip the scheme so both work.
    const storageRef = xml_file.replace("fcode.storage://", "");
    const xmlString = await streamToString(await fcode.storage.download(storageRef));
    const doc = new XMLParser({ ignoreAttributes: false }).parse(xmlString);

    // 2. Fetch employees ONCE, index by lowercased email (email + login_email).
    const { createFactorialClient } = fcode.import("factorial-sdk");
    const factorialClient = createFactorialClient();
    const employees = await factorialClient.employees.employees.all();
    const emailIndex = buildEmailIndex(employees);

    // 3. Walk the parsed doc, enrich matched nodes, collect unmatched emails.
    //    (fast-xml-parser: repeated elements = arrays, single = objects — walk both.)

    // 4. Rebuild and upload as a Factorial document.
    const enrichedXml = new XMLBuilder({ ignoreAttributes: false, format: true }).build(doc);
    const target = emailIndex.get(upload_to_email.trim().toLowerCase());
    if (!target) throw new Error(`No Factorial employee with email ${upload_to_email}`);

    const { data, error, response } = await factorialClient.documents.documents.create({
      body: {
        public: false,
        space: "employee_my_documents",
        company_id: target.company_id,
        employee_id: target.id,
        // Gotcha: author_id is an ACCESS id, not an employee id.
        author_id: target.access_id,
        is_pending_assignment: false,
        file: new Blob([enrichedXml], { type: "application/xml" }),
        file_filename: "enriched-employees.xml",
      },
    });
    if (error || (response && !response.ok)) throw new Error(/* extract message */);

    return { success: true /* counts, unmatched emails, document id */ };
  } catch (error) {
    // Form-facing process: return a readable failure, don't crash.
    return { success: false, message: `The XML could not be processed: ${extractMessage(error)}` };
  }
}
```

SDK error payloads nest unpredictably (e.g.
`{ errors: { errors: ["Employee not found"] } }`) — extract messages by
drilling through `errors`/`message`/`error` keys recursively and flattening
arrays before showing them to the user.

**Patterns:** form file field → `fcode.storage://` reference →
`storage.download` stream · `@add-package` for npm dependencies ·
prefetch-and-index instead of per-item API lookups · SDK file upload via
`Blob` + `file_filename` · `author_id` = `employee.access_id` · top-level
try/catch returning `{ success: false, message }` for form-facing processes.
