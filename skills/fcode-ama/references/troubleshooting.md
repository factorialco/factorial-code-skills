# Troubleshooting the platform journey

Common questions and pitfalls, by journey stage. For code-level problems
(process errors, CLI usage, form embedding), route to the owning skill
instead — see the routing table in `SKILL.md`.

## Access & team

- **"I requested access and heard nothing."** Requests are reviewed by an
  administrator; the outcome arrives as a notification. If it's been
  unreasonably long, escalate (internal: `#factorial-code-users`).
- **"My colleague can see apps I can't."** Apps belong to the development
  team, not the individual. If you're not seeing the team's apps, you may
  have been placed in a different team — escalate to have it corrected.

## App creation & setup

- **"Should I enable the integrations framework?"** Only when the app pushes
  a supported capability to an external system and needs each sync's outcome
  surfaced in Factorial — the decision rule is stage 2 in `journey.md`; probe
  the use case before the user checks it.
- **"There are no OAuth steps in my checklist."** An app that requests no
  OAuth scopes never runs the OAuth flow, so the steps are hidden. Request
  scopes from the App settings' OAuth tab and the setup steps reappear.
- **"I'm a Partner/IC and have no OAuth application."** It's preconfigured by
  an administrator and appears on the App settings' OAuth tab once done — if
  it's blocking you, escalate.
- **"My integrations-framework app doesn't show up / installs oddly."**
  Framework apps must be linked to their Factorial integration (checklist
  step "Link the Factorial integration", on the app's Settings page) —
  installations and the marketplace resolve the app by it.
- **The Getting started checklist disappeared.** It only shows for live
  (`active` / `published`) apps; suspended or archived apps are managed from
  the Settings page.

## Local development

- **"Where do I start coding?"** The Development tab's **"How to build
  locally"** guide gives the two commands (install the CLI, `fcode clone
  <dev-workspace-slug>`); the clone includes the base app code (API clients,
  webhook/schedule/email/form helpers). Everything CLI: `fcode-cli` skill,
  `/docs/cli/`.
- **"`fcode clone dev-<id>` can't find the workspace."** The slug is an
  encoded token, not the app's UUID — copy the exact command from the
  Development tab's "How to build locally" guide (`fcode-cli`).
- **"How do I get all my team's apps at once?"** `fcode team:clone` — it lists
  your development teams when run without an id, then creates one folder per
  app with that app's dev workspace inside. `fcode team:pull` afterwards picks
  up apps added since, and `fcode team:status` reports them all (`fcode-cli`).
- **"`fcode team:pull` says this isn't a development team folder."** Run it at
  the root created by `fcode team:clone` — the one holding `.fcode/team.json` —
  not inside an app or its workspace.
- **"I pushed but the released form/webhook didn't change."** Expected:
  consumers pin to the `stable` alias, and `fcode push` only updates the
  working copy. The released version changes when a new release moves the
  alias. See `fcode-core-concepts`.

## Demo companies & Test marketplace

- **"I can't create a demo company, only request one."** Direct creation
  (and the Demo generator) is for Factorial internal users; Partners and
  Individual Contributors request one and get an email when it's provisioned.
- **"My app isn't listed in the Test marketplace."** It lists the apps in
  *your team's* development environment — check you're in the right team and
  the app is live.
- **"The install asks for credentials."** That's the point: the Test
  marketplace runs the real OAuth flow against the demo company, using that
  demo company's access credentials.
- **"My development OAuth doesn't work."** A development OAuth application
  linked to a demo environment authorizes against **that demo environment's
  API host** — check the OAuth app is linked to the demo company you're
  actually testing with.
- **"OAuth works for one company but fails for others."** Check how the
  Factorial OAuth application was created: a **company-scoped** app (created
  from `/oauth/applications` on the Factorial API host) only ever authorizes
  its own company, while a **global** app (created from the Factorial
  Backoffice) works across companies. An app installed by many companies
  needs a global OAuth app — see stage 4 in `journey.md`.
- **Two `INSTALL`/`SETTINGS`/`UNINSTALL` forms fight each other.** The
  platform doesn't enforce one process per `appRole`; on a clash the
  first process slug (alphabetically) wins and a duplicate-role warning is
  reported. Keep each role on exactly one process.
- **"My installation is stuck in `configuration_pending`."** The install
  isn't finished until setup completes (the `INSTALL` form / setup process).
  Re-open the app from the marketplace and finish configuration.
- **"Variables differ between companies."** By design: each installation
  workspace holds only that company's variables and executions. Shared
  values belong in the parent (dev/prod) workspace; per-company overrides in
  the installation. See `fcode-core-concepts` for inheritance rules.
- **"Where do I see an installation's logs / variables / schedules?"** In its
  `deploy-` workspace — the "Open on platform" action lands on its executions
  page, from the Installations page rows or (beside the company selector) the
  Test marketplace app page.
- **"How do I get a `FACTORIAL_TOKEN` for an installed company?"** Use
  **Copy FACTORIAL_TOKEN** in the "…" menu on the Test marketplace app page.
  It's a live API credential for that company: put it in
  `variables.local.env`, never commit or log it (procedure in `fcode-cli`).
## Releases

- **"Version must be greater than the latest deployed release."** Releases
  are strict semver, always increasing. Note a **failed release keeps its
  version reserved** — resubmit with a higher version, not the same one.
- **"I can't request a release."** The app must be `active` or `published`;
  suspended/archived apps can't release.
- **"My release failed validation."** Fix what the notification lists and
  request a new, higher version; the gate's criteria are in `journey.md` §8.
- **"Who promotes my release to production?"** Requesting a release is open
  to any member; promoting it (pushing to prod and marking it deployed) takes
  a **Factorial admin of the app's owning team** — self-service, the App
  detail page's "How to promote" dialog has the commands — or an **operator**
  for everyone else. See `journey.md` §8 and `/docs/building-apps/permissions`.
- **"I'm a team admin but my push into `prod-…` is refused."** Prod write is
  granted when the release is requested, and access rides in the access
  token — a session opened before the grant doesn't have it. Run
  `fcode login` (or sign out and back in) and retry.

## Publication & marketplace

- **"My app isn't in the production marketplace."** Checklist: (1) it needs a
  deployed release (`published` status); (2) if it's **private**, it's only
  visible to its allowed companies — for anyone else it doesn't exist; (3)
  the listing needs metadata (fallback description or linked DatoCMS record).
- **"I edited the Publication tab but the listing shows something else."**
  A linked DatoCMS record overrides fallback metadata **field by field** —
  edit the DatoCMS record (or unlink it) for those fields.
- **"How do I install a private app for one customer?"** Create a production
  installation from the App detail page, providing the Factorial company ID —
  developers can do this themselves (an administrator can too).
- **"Customers can't connect / authorize."** The customer connect flow (the
  "Connect your Factorial account" page linked from App detail) needs a
  **production OAuth application** and at least one deployed release; until
  both exist it isn't available.
- **"Invalid or expired OAuth state" / "OAuth state missing required
  claims"** on the connect flow: the authorization attempt went stale or was
  tampered with — restart from the connect link.
- **"We unpublished / made the app private — did existing customers lose
  it?"** No: companies with an existing installation keep access.
