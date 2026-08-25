# The Factorial Code journey, stage by stage

Roles: **Internal** (Factorial developer), **Partner** (external company
building with Factorial), **Individual Contributor / IC** (building their own
app or automation; may be a customer, but not always). Where a stage has no
role callout, it is the same for everyone.

## 1. Landing and access

Entry point: `https://code.factorialhr.com` — the public docs and the
request-access form.

Submitting the form opens an **access request** that an administrator reviews
(`pending_approval → approved | rejected`). The requester is notified of the
outcome and, once approved, can sign in to the platform. Describe only that
outcome — never how sign-in credentials are provisioned or delivered.

Approved users belong to a **development team**. Teams share everything: every
app any member creates is visible to and editable by the whole team. If the
administrator sees the requester's organization already has a team on the
platform, the new user is added to that team instead of getting a new one.

## 2. Create an App

From the **Apps** page — creating an app is the first thing to do after
signing in. The creation flow asks for:

1. **Basics** — name and purpose, and the programming language:
   **JavaScript** or **Python** (this selects the language-matched base
   workspaces the app inherits).
2. **OAuth scopes** — the scopes the app will request on the Factorial API.
   They bound the app's permissions, so request only what the app needs.
   Scopes can be extended later from the app settings' OAuth tab, which
   re-opens the corresponding setup step.
3. **Integrations framework** — whether the app uses it. **Steer users away
   from checking this unless they already know what the framework provides**:
   plenty of apps sync data with external systems without it, and opting in
   changes what they build. It applies when the app pushes one of the
   supported capabilities — the Factorial data types it can sync are
   **employee compensation, expenses, and employee updates (for example,
   leaves)** — from Factorial to an external system. Its key benefit is the
   feedback end users get on every sync — success, errors, or a missing
   configuration that needs attention — surfaced right in Factorial instead
   of failing silently. Framework apps inherit the `base-integration-app`
   templates (the generic `OutboundSync` orchestrator), so the developer
   implements only the mapping. Platform docs:
   `/docs/building-apps/integrations-framework/`; full specification:
   `https://apidoc.factorialhr.com/docs/integrations-framework`. A standard
   app is free-form: forms, scheduled jobs, webhooks, reports, syncs of any
   other shape.

App lifecycle: `active → published → suspended → archived` (`published` is a
side effect of the first deployed release, never set by hand; suspended and
archived apps are managed from the app's Settings page).

## 3. The Getting started checklist

The App detail page shows a **Getting started** checklist while the app is
live — the ordered setup steps before customers can install it:

| Step | Shown when | What it is |
|---|---|---|
| **Build the App locally** | always (guidance) | Install the CLI and clone the dev workspace — see stage 5 |
| **Link the Factorial integration** | integrations-framework apps only | The Factorial integration this app maps to; installations and the marketplace resolve the app by it. Done from the app's Settings page |
| **Configure development OAuth** / **Configure production OAuth** | only when the app requests scopes | Client credentials per environment — see stage 4 |
| **Publish a release** | always | See stage 8 |
| **Add marketplace metadata** | always | See stage 9; complete once the listing has a fallback description or a linked DatoCMS record |

An app that requests no scopes never runs the OAuth flow, so the OAuth steps
don't appear; requesting scopes later brings them back.

## 4. OAuth applications

Each environment (development / production) has its own OAuth application —
the client credentials for the OAuth flow against the Factorial API. The
token obtained in the development flow lands in the dev workspace; the
production one is what customers use to authorize their Factorial account.
Docs: `/docs/building-apps/oauth/`.

OAuth lives on the **App settings → OAuth tab**: the requested scopes,
followed by the development and production credentials. The Getting started
checklist and the Dev Marketplace's "Configure dev OAuth" both link there.

On the Factorial side there are **two kinds of OAuth application**, and the
difference matters when authorization fails:

- **Global OAuth apps** — usable by several companies; created from the
  Factorial **Backoffice** (`/backoffice` on the Factorial API host, e.g.
  `https://api.eu2.demo.factorial.dev/backoffice`).
- **Company-scoped OAuth apps** — tied to a single company; created from
  `/oauth/applications` on the Factorial API host (e.g.
  `https://api.eu2.demo.factorial.dev/oauth/applications`).

An app meant to be installed by many companies needs a **global** OAuth app —
a company-scoped one will only ever authorize its own company.

- **Internal**: create the OAuth application yourself — in the **Factorial
  Backoffice** for a global app, or from `/oauth/applications` for a
  company-scoped one — registering the redirect URIs the credential dialog
  shows, then enter the client credentials in that environment's dialog on
  the OAuth tab.
- **Partner / IC**: an administrator preconfigures the OAuth application; it
  appears on the OAuth tab once done. (Automatic creation is planned.)

Once running, tokens are handled by the platform — installed apps get their
OAuth tokens automatically; there is no credential handling in app code.

## 5. Build locally

The recommended loop is local-first, with your own AI agent and the
`fcode-*` skills:

```
pnpm install -g @factorialco/fcode-cli
fcode clone <dev-workspace-slug>
```

Copy both commands from the **"How to build locally"** guide on the app's
Development tab — it fills in the real workspace slug, whose `dev-…` token is
an encoded value, not the app's UUID (see `fcode-cli`).

The clone brings the whole skeleton: the workspace layout, and the shared
base code every app inherits — Factorial API clients, helpers for webhooks,
schedules, email, and forms. Implement and test locally (`fcode run
<process-slug>`), then `fcode push` to upload to the cloud **dev workspace**
and continue integration testing there. CLI reference: `fcode-cli` skill and
`/docs/cli/`.

## 6. Demo companies

Demo companies simulate Factorial customers for the testing phase. They are
managed on the **Demo Companies** page and belong to the development team.

- **Internal**: create demo companies directly using the **Demo generator**,
  and save them in Factorial Code.
- **Partner / IC**: use **Request a demo company**; an administrator
  provisions it and you are notified by email when it's ready.

## 7. Dev Marketplace — test the install

The **Dev Marketplace** is a simulation of the production Factorial
marketplace that runs against demo companies. It lists the apps in the
team's development environment. From there you:

1. Pick the app and the demo company.
2. Go through the real **OAuth flow** with that demo company's access
   credentials.
3. Perform the **install**.

Installing creates a new **installation workspace** (`deploy-{installationId}`)
that inherits from the app's dev workspace. It holds only that company's
variables and executions, fully isolated from other companies — and because
it inherits, any change pushed to the dev workspace is automatically
available in every installation.

Installation lifecycle: `configuration_pending → active → suspended →
deprovisioned` (`active` once the customer completes setup).

For each company that has an installation, the Dev Marketplace app page
links to its `deploy-` workspace (beside the company selector) — that
workspace is where the installation's execution logs, variables, schedules,
and webhooks live. The app's Development and Production tabs show an
installations count linking to the Installations page filtered for that app
and environment.

An installation's "…" menu offers **Copy FACTORIAL_TOKEN** — that `deploy-`
workspace's Factorial API token, for running processes locally in that
company's context (goes in `variables.local.env`; handle as a secret — see
`fcode-cli`). The menu is available on the Dev Marketplace app page (with
Re-install / Uninstall) and on the Installations page (with Suspend).

### appRole forms

The marketplace views render the app's forms according to the `appRole` set
on each process (field reference in `fcode-cli`; embedding in `fcode-forms`):

| appRole | Shown |
|---|---|
| `INSTALL` | At install time — the installation form |
| `SETTINGS` | From the installed app — settings, mappings, configuration |
| `UNINSTALL` | When uninstalling — e.g. to remove webhook subscriptions the app created |
| `USER_FACING_FORM` | As a marketplace utility the installing company's users run — file uploads, one-off automations |

Everything exercised here behaves identically in the production marketplace.

## 8. Release and promotion to production

When development is done, request a **release** from the app's Production
tab: a semver version plus release notes. Any member of the app's team can
request one. The platform enforces that the app is live, the version is
strictly greater than the latest deployed one and unused, and the workspace
snapshot is clean (a snapshot containing failed entities can never become a
release).

The release is then validated — best practices, correct use of the Factorial
Code templates (not reinventing what the base workspaces provide), and
correct handling of secrets and credentials. The release either proceeds or
the user is notified of the changes required; fix them and request again
with a higher version.

Release lifecycle: `requested → deploying → deployed | failed`. Requesting a
release snapshots every process, module, and i18n file into a **pinned
workspace version** in the dev environment and creates the **prod workspace**
(`prod-{appId}`) when it doesn't exist yet. A deployed release:

- has that version copied into the prod workspace (treat its code as
  read-only: changes belong in dev and reach production through a release —
  shared variables and credentials are still managed there);
- points the **`stable` alias** at it, so external consumers roll out (or
  back) by alias moves (mechanics in `fcode-core-concepts`);
- on the first success, flips the app to **published**.

### Who promotes

Promoting the released version into production — pushing it to the prod
workspace and marking the release deployed — takes one of:

- **A Factorial team admin**: a member with a Factorial address holding the
  admin role on the app's owning development team, end to end by themselves.
  Prod workspace write access is granted when the release is **requested**,
  and access rides in the access token — if a push into `prod-…` is refused
  right after requesting, run `fcode login` to pick up the grant. The App
  detail page shows the release actions and a **"How to promote"** dialog
  with the exact `fcode` commands (procedure in `fcode-release`).
- **An operator** (the Factorial Code platform team) — the route for
  Partner / Individual Contributor teams and for members holding the
  developer role.

Marking a release **stable** — re-pointing what every customer installation
follows, i.e. rollout or rollback — takes an operator or **any team admin**,
no Factorial address required: an external team governs its own app's stable
pointer.

The full role-by-workspace matrix is in the public docs:
`/docs/building-apps/permissions`.

The release flow wraps workspace version publishing — don't publish versions
or move `stable` by hand.

## 9. Publication — the marketplace listing

The App detail page's **Publication** tab manages what the listing shows:

- **DatoCMS record** — Factorial Code integrates with DatoCMS, where the
  public meta-information for integrations is managed (descriptions in
  multiple languages, screenshots, tutorials). If the app has a DatoCMS page,
  link it here and the Factorial marketplace consumes it. When both exist,
  DatoCMS content wins over the fallback **field by field**.
- **Fallback metadata** — tagline, markdown description, categories, and a
  screenshot gallery, kept in Factorial Code itself. Enough for apps without
  a public DatoCMS page (e.g. one built for a single customer), where a
  single language suffices.

**Private apps**: a private app is discoverable and installable only by an
allow-list of Factorial companies (it must still be published). The
visibility settings live in **App settings → Configuration**. To make a
private app available to a company, a **production installation** is created
from the App detail page — by an administrator or by a developer who provides
the Factorial company ID (Factorial's Forward Deployed Engineers do this for
customer-specific apps).

## 10. Production

Published apps appear in the production **Marketplace** with the same views
tested in the Dev Marketplace: install (OAuth flow + `INSTALL` form),
settings, user-facing utilities, uninstall. Each installing company gets its
own isolated installation workspace, exactly as in stage 7. The customer
connect flow (production OAuth) is available once the app has a production
OAuth application and a deployed release.
