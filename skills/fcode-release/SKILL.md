---
name: fcode-release
description: Promote a Factorial Code app's code from its dev workspace to its prod workspace — fcode clone the dev- workspace, remote:add the prod- workspace, pull prod's current metadata, then add and push. Use when releasing, promoting, or deploying a Factorial Code (fcode) app's dev workspace to production, or pushing a dev- workspace's code to its prod- counterpart.
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
- **Always confirm before the final `fcode push`.** Summarize what will
  change in the prod workspace and wait for an explicit yes — prod code
  serves live installations.
- **Never `--force`.** If `fcode pull` or `fcode push` reports a divergence
  between local and prod, stop, show what diverged, and ask the user how to
  proceed (`fcode-cli`).
- **The `fcode pull` after `remote:add` is mandatory** — it picks up the
  prod workspace's current metadata. Skipping it pushes stale settings over
  prod's.
- **Both slugs are encoded tokens** (`dev-…` / `prod-…`), not the app's
  UUID (see `fcode-cli`). Verify the prefixes: cloning target and push
  target must not be swapped.
- **A failing `fcode clone`** usually means a mistyped token or missing
  access (see `fcode-ama`) — don't retry blindly.
- **If `dev-xxx/` already exists locally, don't reuse it silently** — ask
  whether to release from a fresh clone (recommended) or the existing copy,
  and `fcode pull` it first.

## Procedure

1. **Collect the two slugs.** One `dev-…` (source) and one `prod-…`
   (target). Wrong prefix or any ambiguity about which app they belong to:
   stop and ask.
2. **Validation gate.** Ensure a current ✅ `APP_VALIDATION_REPORT.md`
   exists for the dev workspace — run `fcode-code-validation` when it is
   missing or stale. On ❌, list the Blockers and refuse; proceed only on
   an explicit user override.
3. **Clone and wire the prod remote:**

   ```sh
   fcode clone --skipSkillsSetup dev-xxx
   cd dev-xxx
   fcode remote:add prod-xxx
   fcode pull          # mandatory: picks up prod's current metadata
   fcode add           # registers resources prod doesn't have yet
   ```

   On a divergence reported by `fcode pull`: stop and ask (see Gotchas).
4. **Confirm.** Run `fcode status`, summarize for the user what the push
   will change in `prod-xxx` (processes, modules, variables, settings), and
   wait for explicit confirmation.
5. **Push:**

   ```sh
   fcode push
   ```

   On a divergence: stop and ask — never `--force` on your own.
6. **Report the outcome**, reminding the user that the platform release
   (publish a version, move `stable`) is a separate step (`fcode-ama`).
