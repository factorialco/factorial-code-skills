---
name: fcode-release
description: Promote a Factorial Code app's code from its dev workspace to its prod workspace — clone the dev- workspace, remote:add the prod- workspace and pull, put dev's code back over what the pull wrote, then push and delete what dev removed. Use when releasing, promoting, or deploying a Factorial Code (fcode) app's dev workspace to production, or pushing a dev- workspace's code to its prod- counterpart.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Release

Promotes an app's code from its `dev-…` workspace to its `prod-…` workspace
using CLI remotes. Input: the two workspace slugs. This updates prod's
**current** (unversioned) code only — publishing a workspace version and moving
the `stable` alias is the separate platform release (model in
`fcode-core-concepts`, journey in `fcode-ama`): don't run `settings:versions:*` or
move `stable` in this flow unless explicitly asked.

Gate every release with `fcode-code-validation` — see the procedure.

## Gotchas

- **Prod write access is required.** Only operators and the app's Factorial
  team admins can push to a `prod-…` workspace; the admin's grant lands when
  a release is requested and rides in the access token — if prod access is
  refused right after requesting, run `fcode login` and retry. Who promotes
  and why: `fcode-ama`.
- **A ✅ validation is required before pushing.** Run `fcode-code-validation`
  on the dev workspace first; while the report has Blockers, refuse to push.
  Only an explicit user override ("push anyway despite the Blockers")
  proceeds — record that the gate was overridden.
- **Always confirm before pushing.** Summarize what will change in the prod
  workspace (uploads, creations and deletions) and wait for an explicit yes
  — prod code serves live installations.
- **Never a blanket `--force`.** Not on `fcode push`, `fcode pull`, or a
  `<kind>:push` without a slug. The one allowed `--force` deletes a single
  resource: `fcode processes:push <slug> --force` (or `modules:push <slug>`,
  `i18n:push <locale>`). Use one slug per command, only for a slug in the
  deletion list the user confirmed, and only while
  `fcode <kind>:status <slug>` shows it `🗑️💻 (removed-local)` — on any other
  status the same command overwrites prod. Any other divergence between
  local and prod (a `🔥 (conflict)`): stop, show it with `fcode diff`, and
  ask (`fcode-cli`).
- **The prod pull overwrites dev's code.** The first `fcode pull` after
  `remote:add` saves every process, module, own locale and dependency
  manifest prod has over the local files (logged `[✨ new]`), and
  `fcode status` then reads them as up to date, so a push would release
  nothing. The pull is still mandatory: it records prod's ids, and without
  them every push of a slug prod already has fails as a duplicate. So the
  procedure puts dev's code back from a reference copy after the pull
  (`fcode-cli` §`fcode remote:add`).
- **Only dev's own live resources are dev's code.** The clone also writes,
  read-only, what dev inherits (`🔗 <parent>` in `*:status --showInherited`)
  and what dev deleted but a workspace version keeps (`🪦 kept by <tags>`).
  A row with neither mark is dev-owned live. Nothing else may reach prod: if
  prod doesn't have one of these, `fcode add` would register it and
  `fcode push` would create it.
- **Workspace settings and variables are not copied from dev.** Never
  restore `settings.json` or the variables files from the reference: prod
  keeps its own values. A key prod doesn't own keeps dev's line in
  `variables.env` after the pull, and the push would create it in prod with
  that value (the `********` placeholder for a sensitive one): step 6
  settles each one. Customers read the App's description for Factorial
  One (`factorial` in `settings.json`) from prod: if prod's is missing or
  stale, edit it after the pull and before the push, and include it in the
  confirmation summary (`fcode-cli` §Workspace settings).
- **Both slugs are encoded tokens** (`dev-…` / `prod-…`), not the app's
  UUID (see `fcode-cli`). Verify the prefixes: cloning target and push
  target must not be swapped.
- **A failing `fcode clone`** usually means a mistyped token or missing
  access (see `fcode-ama`) — don't retry blindly.
- **Always release from a fresh clone in a new, empty folder outside any
  git repository**, even when `dev-xxx/` or a development team's `app/`
  checkout already exists, and never `remote:add` prod in an existing copy.
  The folder holds a copy pointed at prod and prod's variables: delete it
  when the release ends or stops. An existing copy's old prod sync record
  hides what changed in prod, and a checkout left pointing at prod sends
  every teammate's push to production (`fcode-core-concepts` §Workspace
  structure (CLI)). Unpushed changes in an existing copy are not in dev and
  not validated, so they are not released: tell the user.

## Procedure

1. **Collect the two slugs.** One `dev-…` (source) and one `prod-…`
   (target). Wrong prefix or any ambiguity about which app they belong to:
   stop and ask.
2. **Validation gate.** Ensure a current ✅ `APP_VALIDATION_REPORT.md`
   exists for the dev workspace — run `fcode-code-validation` when it is
   missing or stale. On ❌, list the Blockers and refuse; proceed only on
   an explicit user override.
3. **Clone dev, and keep an untouched copy as the reference:**

   ```sh
   cd "$(mktemp -d)"                      # new, empty folder
   git rev-parse --show-toplevel          # must fail: no repository here
   fcode clone --skipSkillsSetup dev-xxx
   cp -R dev-xxx dev-ref                  # dev's code; never re-pointed
   cd dev-ref
   fcode processes:status --showInherited
   fcode modules:status --showInherited
   fcode i18n:status --showInherited
   cd ..
   ```

   Note dev's own live processes, modules and locales: the rows with no
   `🔗` and no `🪦`.
4. **Wire prod and pull:**

   ```sh
   cd dev-xxx
   fcode remote:add prod-xxx
   fcode pull          # mandatory; overwrites what prod has with prod's code
   ```

   - On a divergence reported by `fcode pull`: stop and ask.
   - Run the same three `status --showInherited` commands here to note
     prod's own live resources, and `fcode status` for its "only existing
     in local" lines.
   - If a dev-owned live process or module shows in prod as `🔗` or `🪦`,
     stop and ask: this flow can't replace it. A dev-owned locale next to
     one prod inherits (`🔗` in that locale's column) is not a blocker: it
     is an override to create.
5. **Put dev's code back:** every dev-owned live process and module without
   its `versions/` folder (prod's read-only snapshots stay), dev's own
   locale files (never `*.inherited.yaml`), and each own dependency
   manifest `dev-ref` has (never `*.inherited.*`):

   ```sh
   rsync -a --checksum --delete --exclude versions/ \
     ../dev-ref/processes/<slug>/ processes/<slug>/
   rsync -a --checksum --delete --exclude versions/ \
     ../dev-ref/modules/<slug>/ modules/<slug>/
   cp ../dev-ref/i18n/<locale>.yaml i18n/
   cp ../dev-ref/dependencies/package.json dependencies/
   cp ../dev-ref/dependencies/requirements.txt dependencies/
   ```

   Never `settings.json` or the variables files. A manifest prod owns that
   dev doesn't can't be deleted: report it in step 7.
6. **Remove what must not reach prod, settle variables, then register what
   is new:**
   - **Leftovers:** a process, module or locale in the "only existing in
     local" lines that `dev-ref` shows as `🔗` or `🪦`. Delete its folder
     (`rm -rf processes/<slug>`, `modules/<slug>`) or file
     (`rm i18n/<locale>.yaml`). One that is neither dev-owned live nor
     `🔗`/`🪦` in `dev-ref`: stop and ask. Variables and manifests
     (`JAVASCRIPT`, `PYTHON`) in those lines are not leftovers.
   - **Variables:** each key in the variables "only existing in local"
     line is dev's line, not dev's code:
     - prod inherits it (it is in `variables.inherited.env`): remove dev's
       line from `variables.env`, unless the user asks for a prod override;
     - sensitive (`variables.meta.json`): its value is the `********`
       placeholder, and a push would create a prod secret with that
       literal value that no later push can change. Remove the key from
       `variables.env` and `variables.meta.json`, and ask the user to
       create it in prod;
     - otherwise ask the user for prod's value and write it, or remove the
       key from both files.
   - **Deletions:** each process, module and locale prod owns live that
     dev doesn't, sorted by what `dev-ref` shows:
     - `🪦`: deleted in dev;
     - no trace: deleted in dev, or it only ever existed in prod (a
       hotfix, an operator's process) — the user confirms each one by
       itself;
     - `🔗`: dev's parent now provides it, and prod only inherits it if
       prod's parent does too: stop and ask, and keep it until the user
       confirms prod's parent has it.

     If prod's `settings.json` names a slug on the list (such as
     `errorHandlerConfig.processSlug`), ask before going on. Delete each
     the same way as a leftover. This is the deletion list.

   ```sh
   fcode add           # registers dev's new resources prod doesn't have yet
   ```

7. **Confirm.** Run `fcode status` and summarize for the user what will
   change in `prod-xxx`:
   - uploads (`🔺 (modified)`): dev's version replaces prod's. Read the
     `-` (prod) side of `fcode diff` and warn about any change only prod
     has: the upload loses it;
   - creations (`✨ (new-local)`), with the values step 6 settled for
     variables new to prod, and each prod override of an inherited
     variable the user asked for (shown `🔥 (conflict)`);
   - deletions (`🗑️💻 (removed-local)`), listing apart the ones that may
     exist only in prod — one a prod workspace version contains stays
     there as deleted;
   - manifests prod keeps although dev has none;
   - settings changes.

   Wait for an explicit yes. If the user doesn't confirm a deletion, bring
   it back with `fcode <kind>:reset <slug>`, always with the slug: without
   it, reset also overwrites every restored (modified or conflicting)
   resource with prod's code and drops the added-but-unpushed ones.
8. **Push, then delete one slug at a time:**

   ```sh
   fcode push </dev/null
   fcode processes:push <slug> --force    # per deleted process
   fcode modules:push <slug> --force      # per deleted module
   fcode i18n:push <locale> --force       # per deleted locale
   ```

   - `fcode push` uploads and creates. With stdin closed it answers no to
     each pending deletion, reports those slugs as skipped and exits `1`
     (`fcode-cli` §`fcode push`), which is expected for the confirmed
     deletions only. Any other skipped slug: stop and ask. Never pipe
     `yes` into it.
   - Then run the deletions: processes first, modules after (a module a
     prod process still imports can't be deleted), locales last. Run each
     only once `fcode <kind>:status <slug>` shows `🗑️💻 (removed-local)`.
   - Each deletion prints `[🗑️  removed]` and exits `0`.
     `📌 '<slug>' is kept as deleted: …` means a prod workspace version
     holds it, which is expected.
   - A failed upload or deletion prints `🚨` without changing the exit
     code (`fcode-cli`). Read the output; on any `🚨`, stop and report.
9. **Check prod matches dev:**

   ```sh
   fcode pull          # brings kept-as-deleted ones back read-only
   fcode diff          # exits 0 when local and prod match
   diff -r -x versions -x parameters.json \
     ../dev-ref/processes/<slug> processes/<slug>
   ```

   - `fcode status` must list no change and no "only existing in local"
     line.
   - The `diff -r` must print nothing for every dev-owned live process,
     and the same for modules, dev's own locale files and manifests: a
     difference is dev code that didn't reach prod.
   - Every slug on the confirmed deletion list is gone or `🪦` in
     `*:status --showInherited`.
10. **Report the outcome:** uploads, creations, deletions (and the ones
    kept as deleted). Remind the user that the platform release (publish a
    version, move `stable`) is a separate step (`fcode-ama`). Delete the
    temporary folder.
