---
name: fcode-release
description: Release a Factorial Code app to production — validate it, then request the release with `fcode app:releases:request <version>` (or the console's Request release dialog), which copies the dev workspace into prod; follow it with `fcode app:releases:list`. Use when releasing, promoting, or deploying a Factorial Code (fcode) app's dev workspace to production, requesting or checking a release, or pushing a dev- workspace's code to its prod- counterpart.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — Release

A release copies an App's `dev-…` workspace into its `prod-…` workspace. You
**request** it; the dashboard does the copy. Nobody pushes code to `prod-…` by
hand: no `remote:add`, no clone pointed at prod.

Requesting a release, for version `X`:

1. tags the dev workspace with `X` (a workspace version, see
   `fcode-core-concepts`);
2. copies that version into `prod-…` — created on the App's first release —
   and deletes from prod what dev no longer has;
3. tags prod with `X`.

The copy runs in the background and retries on its own when the platform
fails; when it gives up, the release turns `failed` with the error. Once
copied, the release stays `requested` until an operator or a Factorial admin
of the team marks it `deploying`, `deployed` or `failed`. Who does what:
`fcode-ama`.

Gate every release with `fcode-code-validation` — see the procedure.

## Gotchas

- **A ✅ validation is required before requesting.** Run
  `fcode-code-validation` on the dev workspace first; while the report has
  Blockers, refuse to request. Only an explicit user override ("release
  anyway despite the Blockers") proceeds — record that the gate was
  overridden.
- **The release copies the cloud, not your files.** It copies dev as it is in
  the cloud. Changes only in a local checkout are not released: push them to
  dev (`fcode push`) and validate again first.
- **Variables and workspace settings are never copied.** The copy takes
  processes, modules, locales and dependency manifests. A variable dev owns
  that prod lacks must be set in prod by hand, in the prod workspace's
  variables page; the request lists them, so tell the user before they
  confirm. Settings such as the App's description for Factorial One are
  prod's own: edit them in the prod workspace's **Settings › Factorial** tab
  (`fcode-cli` §Workspace settings).
- **The copy overwrites and deletes in prod.** Dev's version of every
  process, module and locale replaces prod's, so a fix made only in prod is
  lost: port it to dev first. Every one prod has and dev doesn't is deleted,
  including a hotfix that only ever existed in prod. The request lists the
  deletions, not the overwrites; read the list to the user before they
  confirm.
- **Prod's settings can name a deleted process.** Settings are not copied, so
  prod's can still point at a process on the deletion list (an error
  handler, `errorHandlerConfig.processSlug`). When the list has one, ask the
  user to check the prod workspace's Settings before confirming.
- **"Could not check production" means unknown, not empty.** When the request
  cannot read prod, it says so and still lets you request: the deletions and
  missing variables are unknown. Retry the dry run; request only if the user
  confirms knowing that.
- **One release in progress per App.** While a release is `requested` or
  `deploying`, a new request is refused. An operator or a Factorial admin of
  the team marks it `deployed` or `failed` first.
- **Versions are semver, without `v`,** and higher than the last deployed
  release (`1.4.0`, `1.4.0-rc.1`). A version already used is refused, except a
  `failed` one, which can be requested again once its cause is fixed.
- **Always confirm before requesting.** Summarize what the release changes
  in prod (the deletions and the missing variables) and wait for an explicit
  yes — prod code serves live installations.

## Procedure

1. **Find the App folder.** The release commands run from a development
   team checkout (`fcode team:clone`, see `fcode-cli`): the App's folder, or
   anywhere inside its `app/`. Without one, clone the team first, or have the
   user use the console's **Request release** dialog (App page → Production
   tab), which shows the same checks.
2. **Validation gate.** Ensure a current ✅ `APP_VALIDATION_REPORT.md` exists
   for the dev workspace — run `fcode-code-validation` when it is missing or
   stale. On ❌, list the Blockers and refuse; proceed only on an explicit
   user override.
3. **Check the state:**

   ```sh
   fcode app:releases:list
   ```

   A release still `requested` or `deploying` blocks a new one: stop and tell
   the user who can close it. Pick the version: the next one above every
   release listed, as the user wants it (patch, minor or major), or the
   version of a `failed` release to retry it.
4. **Preview:**

   ```sh
   fcode app:releases:request <version> --notes "<what changed>" --dry-run
   ```

   It prints what the copy does, and requests nothing:
   - **deleted from production**: processes, modules and locales prod has and
     dev doesn't;
   - **missing in production**: variables to set in prod by hand, with the
     link to prod's variables page;
   - **your local `app/` differs from dev**: local edits, or dev changes not
     pulled yet. Only the first matters, and they are not released;
   - **could not check production**: see the gotcha above.

   `Last change in dev` says when dev last changed and who changed it: check
   it matches what the user expects to release.
5. **Confirm.** Show the user the version, the notes and the lists from
   step 4, and wait for an explicit yes.
6. **Request:**

   ```sh
   fcode app:releases:request <version> --notes "<what changed>" --yes
   ```

   The command requests the release, waits up to five minutes for the copy,
   then pulls `app/` so the new version appears under each `versions/`. It
   ends in one of three ways:
   - `✅ Release X requested and copied to production` (exit `0`): done.
   - `⚠️ … copy to production is still running` (exit `0`): the copy keeps
     retrying in the background. Follow it with `fcode app:releases:list`
     (`requested (copying)` until it ends), then `fcode pull` in `app/`. If
     it ends `failed`, the table does not show why:
     `fcode app:releases:list --json` has it in `error`.
   - `Release X failed: its copy to production gave up (<error>)` (exit
     `1`): read the error, fix the cause, and request the same version again.

   `--json` (with `--yes` or `--dry-run`) prints the same as one JSON object:
   `copy` is `copied`, `running` or `failed`, `followError` says why the
   command stopped following the copy (exit `1`), and `pulled` whether `app/`
   was refreshed.
7. **Report the outcome:** the version, what was deleted from prod, and the
   variables the user still has to set there. Remind the user that the
   release awaits being marked `deployed` by an operator or a Factorial admin
   of the team. Customers' installations follow the `stable` alias: only the
   App's first deployed release becomes stable on its own; a later one
   reaches them when an operator or a team admin moves `stable` to it
   (`fcode-ama`).

## Errors

- **`Release X is still requested/deploying`**: another release is in
  progress (step 3).
- **`Version X is not higher than Y`** or **`already exists`**: pick another
  version; `fcode app:releases:list` shows the ones in use.
- **`You are not a member of this App`**: the user's session predates their
  team membership — `fcode login`, then retry.
- **`This App cannot be released`**: only an `active` or `published` App can
  be released; a suspended or archived one cannot.
- **`The dashboard does not know this App`**: the team folder is out of
  date — `fcode team:pull`, then retry.
- **`app/ kept N local change(s) the pull would overwrite or remove`**: the
  release went through; the user runs `fcode pull` in `app/` to choose.
